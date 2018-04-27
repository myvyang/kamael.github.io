## 高性能动态爬虫-Chromium改造实录

## 开篇

一个黑盒的漏洞扫描器包含很多部分：爬虫，URL管理，POC管理，检测等。而其中，所有的爬虫的运行逻辑都基本类似：输入一个启始URL，请求，然后从返回中提取出尽可能多的新URL。如此循环。

对于一个大型站点，如何尽可能多的发现新URL，直接决定了漏洞产出的数量。使用动态容易来渲染页面然后提取URL显然比正则匹配能够拿到更多的结果。但是动态爬虫带来的问题是，爬取同样数量的页面，需要部署的机器可能增长几十倍。

为了降低成本，需要一个高性能(在相同资源的情况下，单位时间能够爬取更多的页面)的爬虫容器。为了达到这一目的，经过权衡，最终选择了直接在原生Chromium基础上，进行深度定制。定制后的Chromium，相同机器配置下，爬取速度实测至少为原生的15-20倍。

## 整体设计

一般的动态爬虫，都只是把渲染容器(phantomjs，chrome等)作为一个被调用组件，每次需要渲染页面时，传入一个URL，然后取出渲染后的DOM，从中提取URL。

这样做固然有很多好处，但是性能上存在大量损失，一方面，一个浏览器进程只能渲染一个页面；另一方面，和浏览器的通信，以及DOM的序列化和重解析等都浪费了时间。(这里假定一般的渲染容器也不会频繁的销毁和创建浏览器进程，否则又是一大笔开销)

在构建爬虫时，采用了如下架构：

![arch](https://static.dingtalk.com/media/lADPBbCc1ak50oXNBCLNBQo_1290_1058.jpg)

首先外部启动定制的Chromium的时候，通过参数设置启主页面`init.html`，同时加载一个自己编写的浏览器拓展`spiderInjection`。

这个架构最大的特色在于：__所有的URL队列管理，渲染，页面信息和URL收集都通过这一个开始页面和浏览器拓展完成。__ 省去了所有的和外部的通信开销，爬虫本身的实现包含了完整的逻辑。同时，在启动浏览器时，参数加上`--single-process`设置为单进程模式，以降资源占用降到最低。

容器启动后，`init.html`内包含16个iframe子页面，这些子页面将会执行渲染页面的任务。浏览器拓展`spiderInjection `会分别向主页面`init.html`和子页面注入一段JS，主页面JS负责URL队列的管理和远端的交互，子页面JS负责在页面渲染完后收集页面中提取到的所有URL。

一开始URL队列为空，会不断的请求远程接口，获取种子URL，然后加载到一个iframe中进行渲染。渲染完成后，子页面JS将提取URL加入队列中，由主页面调度到其他iframe内进行渲染，如此循环。

要实现这样一种爬虫模式，需要

1. 保证iframe内的页面不会探测到自己在iframe中，及运行在隔离环境中(这里暂不考虑页面为了防爬虫，特定做严格的浏览器环境检测，只考虑常规的iframe检测)。

  iframe的sandbox属性可以部分满足，但是测试中发现，sandbox的一些安全特性会导致某些页面加载不正常。我们[改造了子页面逻辑](#改造子页面逻辑)，让iframe内的页面认为自己运行在顶级页面中。

2. 保证容器可以实现自循环，不需要外界干预。

  由于所有页面在同一个tab页中渲染，任何一个页面被阻塞会导致容器罢工。这个问题的解决见[避免阻塞](#避免阻塞)

3. 尽可能全的提取页面内可能包含的所有URL。包括常规的链接，ajax请求，通过交互暴露的链接和ajax请求，form表单，JS通过`window.open`打开的页面。实现见[页面URL收集](#页面URL收集)

4. 一些性能优化的工作。见[性能优化](#性能优化)

同时，为了辅助上述功能的实现，通过简单的修改，可以将浏览器的C++代码中捕获的变量传递给JS层使用，见[C++和JS通信](#C++和JS通信)

## 改造子页面逻辑

这个架构采用了通过多个iframe子页面加载渲染的方式来提高渲染速度(因为实际中，大多数的渲染时间浪费在了等待网络请求上，实际不占用机器资源)。

为了实现良好的效果，需要对浏览器的window对象的派生链进行改造，让iframe内的JS以为自己运行在顶级页面中，同时我们注入的JS又可以通过派生链来管理这些子页面。

在Chromium中，window对象的派生链是通过`window.parent`实现的。最顶层的window对象的parent是其自身：

```
window.parent === window
window.parent === self
// 顶层页面为true，否则为false
```

修改浏览器代码，让所有的页面的`window.parent`对象指向自身。这里给自己的域名留一个后门，因为需要保持主页面`init.html`的控制权限。

修改`window.parent`的实现：


```
--- a/third_party/WebKit/Source/core/frame/DOMWindow.cpp
+++ b/third_party/WebKit/Source/core/frame/DOMWindow.cpp
 DOMWindow* DOMWindow::parent() const {
   if (!GetFrame())
     return nullptr;
+  
+  Frame* parent = GetFrame()->Tree().Parent();
+  if (parent) {
+    if (parent->DomWindow()->isOurHost()) {
+      return parent->DomWindow();
+      // 如果是自己的特定域名，则返回真实的情况，否则返回window对象自身。
+    }
+  }
+  
+  return GetFrame()->DomWindow();
+}
```

添加一个`window.oparent`实现，以便注入的拓展的JS可以识别自己是否在iframe中：

```
+DOMWindow* DOMWindow::oparent() const {
+  if (!GetFrame())
+    return nullptr;

   Frame* parent = GetFrame()->Tree().Parent();
   return parent ? parent->DomWindow() : GetFrame()->DomWindow();
 }
```

修改`window.top`实现：

```
 DOMWindow* DOMWindow::top() const {
   if (!GetFrame())
     return nullptr;
+  
+  const DOMWindow* top_ = this;
+  if (top_->parent()) {
+    top_ = top_->parent();
+  }

-  return GetFrame()->Tree().Top().DomWindow();
+  return top_->GetFrame()->DomWindow();
+  // return GetFrame()->Tree().Top().DomWindow();
 }
```

因为`window.parent`逻辑已经被hook了，这里直接利用。


## 避免阻塞

为了保障容器能够在无交互的前提下自主运行，需要做一些基础改造，把已知的可能导致浏览器block的情况都规避掉。

已知的有以下几种情况会阻断浏览器加载：

1. `alert`等弹框阻塞。
2. `location.href`跳转。因为需要尽可能全的收集URL，如果一个页面渲染到一半被JS跳转走了，可能有部分URL漏掉了。
3. 如果设置了`onbeforeunload`事件，也会阻塞。
4. 浏览器打开伪协议时，也会block页面。

### 屏蔽弹框

针对`third_party/WebKit/Source/core/frame/LocalDOMWindow.cpp`中的四个函数：`alert`，`confirm`，`prompt`，`print`。

在每个函数的起始添加一行`return`，例如`print`：

```
--- a/third_party/WebKit/Source/core/frame/LocalDOMWindow.cpp
+++ b/third_party/WebKit/Source/core/frame/LocalDOMWindow.cpp
@@ -698,6 +698,8 @@ Element* LocalDOMWindow::frameElement() const {
 void LocalDOMWindow::blur() {}

 void LocalDOMWindow::print(ScriptState* script_state) {
+  return;
+
   if (!GetFrame())
     return;
```

### 屏蔽跳转

在`Location::SetLocation`中阻断。

```
--- a/third_party/WebKit/Source/core/frame/Location.cpp
+++ b/third_party/WebKit/Source/core/frame/Location.cpp
@@ -235,6 +237,16 @@ void Location::SetLocation(const String& url,
                            LocalDOMWindow* entered_window,
                            ExceptionState* exception_state,
                            SetLocationPolicy set_location_policy) {
+  if ((current_window->oparent() != current_window && current_window->oparent()->isOurHost())) {
+    String completed_url = entered_window->GetFrame()->GetDocument()->CompleteURL(url).GetString();
+    ...(TODO)
+    ...
+    return;
+  }
   if (!IsAttached())
     return;
```

注意这里的`(TODO)`，因为跳转的目标URL可能是我们需要收集的，所以这里先记录下来，后面会写这么得到这个目标URL。

### 干掉 onbeforeunload

找到该函数，将内容注释掉即可：

```
--- a/content/browser/frame_host/render_frame_host_impl.cc
+++ b/content/browser/frame_host/render_frame_host_impl.cc
@@ -1985,17 +1985,17 @@ void RenderFrameHostImpl::OnRunBeforeUnloadConfirm
(bool is_reload, IPC::Message* reply_msg) {
  // ...
 }
```

### 干掉伪协议

如果浏览器调用`qq://some_args`这种伪协议，也会弹框导致hang住，直接禁止伪协议：

```
--- a/content/browser/child_process_security_policy_impl.cc
+++ b/content/browser/child_process_security_policy_impl.cc
@@ -666,6 +666,8 @@ bool ChildProcessSecurityPolicyImpl::CanRequestURL(
   if (CanCommitURL(child_id, url))
     return true;

+  return false;  
```

## URL收集

### 页面基本URL收集

对于DOM中的URL，通过拓展注入的JS直接在DOM中进行收集：

```
        // 收集href
        var hrefs = document.querySelectorAll('[href]');
        for (var i = 0; i < hrefs.length; i++) {
            urls.push(hrefs[i].href);
        }

        // 收集src
        var srcs = document.querySelectorAll('[src]');
        for (var i = 0; i < srcs.length; i++) {
            urls.push(srcs[i].src);
        }
        
        // 通过正则，对URL格式的链接进行收集
        var regexs = [
            /href=["']([^"']+)["']/g,
            /src=["']([^"']+)["']"/g,
            /((?:https?:)?\/\/(?:\w+\.)+\w+[^"'\s<>]+)/g
        ];

        for (var i = 0; i < regexs.length; i++) {
            var result = matchAll(source, regexs[i]);
            for (var j = 0; j < result.length; j++) {
                var fullUrl = document.completeUrl(result[j]);
                urls.push(fullUrl);
                break;
            }
            break;
        }
```

### 请求URL收集

部分URL不会直接出现在DOM中，会通过ajax请求触发，这部分直接在拓展的`backgrund.js`中收集：

```
chrome.webRequest.onBeforeRequest.addListener(
    function(details){
        var url = details.url;
        collectUrls.push(url);
    },
    {urls:["<all_urls>"]},
    ["blocking"]
);
```

### 事件触发

上面的步骤只收集到了页面直接渲染得到的URL。还有一些URL需要交互(点击，滑动等)触发JS事件才能收集到。

一直直观的方式是通过遍历DOM节点，不断的触发事件。对于小型的网页还好，但是一些复杂的网页可能有上万个节点，遍历一遍耗时太久。分析Chromium代码后，可以发现，所有的事件绑定都是通过`EventTarget::addEventListener`函数进行的，因此直接hook这个函数，就可以拿到所有的事件，然后逐一触发即可。

```
diff --git a/third_party/WebKit/Source/core/events/EventTarget.cpp b/third_party/WebKit/Source/core/events/EventTarget.cpp
index ae79094..8117196 100644
--- a/third_party/WebKit/Source/core/events/EventTarget.cpp
+++ b/third_party/WebKit/Source/core/events/EventTarget.cpp
@@ -307,6 +307,20 @@ void EventTarget::SetDefaultAddEventListenerOptions(
 bool EventTarget::addEventListener(const AtomicString& event_type,
                                    EventListener* listener,
                                    bool use_capture) {
+  LocalDOMWindow* executing_window = this->ExecutingWindow();
+
+  Node* node = this->ToNode();
+  if (executing_window && node) {
+      executing_window->AddEventNode(event_type, node);
+      
+  }
+
```

这里的`AddEventNode`是我自己添加的window对象方法，这个方法会将拿到的事件类型和对应的绑定节点放进一个自添加的window属性里，这个属性会通过修改IDL暴露给JS层，JS层拿到这个列表后，逐个遍历触发事件。

```
        var firedEventNames = ["focus", "mouseover", "mousedown", "click", "error"];
        // 只关注这五个事件
        var firedEvents = {};

        var length = firedEventNames.length;

        for (var i = 0; i < length; i++) {
            firedEvents[firedEventNames[i]] = document.createEvent("HTMLEvents");
            firedEvents[firedEventNames[i]].initEvent(firedEventNames[i], true, true);
        }

        var eventLength = window.eventNames.length;
        // window.eventNames，window.eventNodes即为添加的window属性，其中eventNames为事件类型，eventNodes为绑定该事件的节点。
        for (var i = 0; i < eventLength; i++) {
            var eventName =  window.eventNames[i].split("_-_")[0];
            var eventNode =  window.eventNodes[i];

            var index = firedEventNames.indexOf(eventName);
            if (index > -1) {
                if (eventNode != undefined) {
                    eventNode.dispatchEvent(firedEvents[eventName]);
                }
            }
        }
```

在触发事件之后，如果页面发生跳转，会导致后面的事件无法触发，也就意味着可能会漏掉一些URL。

有三种情况会导致当前页面发生跳转：

1. 触发的JS中包含`location.href=xxx`这类代码。由于hook了`setLocation`，实际不会跳转。这种情况会产生一条新的URL，同样的，通过添加window属性，会将这条URL在Chromium的C++代码中收集，然后暴露给JS层。
2. 触发事件时，可能会触发表单提交，也会导致页面发生跳转。因此需要hook表单提交相关的代码：

```
diff --git a/third_party/WebKit/Source/core/loader/FormSubmission.cpp b/third_party/WebKit/Source/core/loader/FormSubmission.cpp
index b54218b..2d7851e 100644
--- a/third_party/WebKit/Source/core/loader/FormSubmission.cpp
+++ b/third_party/WebKit/Source/core/loader/FormSubmission.cpp
@@ -157,7 +157,9 @@ inline FormSubmission::FormSubmission(SubmitMethod method,
       form_(form),
       form_data_(std::move(data)),
       boundary_(boundary),
-      event_(event) {}
+      event_(event) {
+    target_ = "formTarget";
+  }
```

这里修改`target_`，等价于修改`form`的`target`属性。把表单提交的target窗口重定向到一个特定的iframe页面，这样就可以避免表单提交将当前页面跳走了，同时，通过浏览器拓展，也能收集到表单提交的请求URL。

3. A标签点击的默认效果是跳转。

```
diff --git a/third_party/WebKit/Source/core/html/HTMLAnchorElement.cpp b/third_party/WebKit/Source/core/html/HTMLAnchorElement.cpp
index cafb870..3dd0c72 100644
--- a/third_party/WebKit/Source/core/html/HTMLAnchorElement.cpp
+++ b/third_party/WebKit/Source/core/html/HTMLAnchorElement.cpp
@@ -340,6 +340,13 @@ void HTMLAnchorElement::HandleClick(Event* event) {
   AppendServerMapMousePosition(url, event);
   KURL completed_url = GetDocument().CompleteURL(url.ToString());

+  if (frame->DomWindow() && frame->DomWindow()->oparent()) {
+    if (frame->DomWindow()->oparent()->isOurHost()) {
+      return;
+    }
+  }
```

这里就直接在跳转前把行为取消掉即可。A标签的URL会通过JS收集到，这里不需要记录。

### 其他收集

收集`window.open`的URL：

```
--- a/third_party/WebKit/Source/core/frame/LocalDOMWindow.cpp
+++ b/third_party/WebKit/Source/core/frame/LocalDOMWindow.cpp
@@ -1594,6 +1600,8 @@ DOMWindow* LocalDOMWindow::open(const String& url_string,
       target_frame = GetFrame();
   }

+  target_frame = GetFrame()->Tree().Find("formTarget");
+
   if (target_frame) {
     if (!active_document->GetFrame() ||
         !active_document->GetFrame()->CanNavigate(*target_frame)) {
@@ -1614,6 +1622,16 @@ DOMWindow* LocalDOMWindow::open(const String& url_string,
     return target_frame->DomWindow();
   }

+
+  String completed_url = first_frame->GetDocument()->CompleteURL(url_string).GetString();
+  this->appendInfo("_-_");
+  this->appendInfo(completed_url);
+  return nullptr;
```

这里首先修改`target_frame`，让`window.open`打开的页面在指定的窗口，避免跳转当前页面。然后将URL通过自己添加的window属性收集起来。

## 性能优化

### 图片

网页中可能代码占的带宽并不大，大部分带宽都用于图片的加载。Chromium自身支持伪图片加载模式(即渲染的时候还是会为图片留出占位符，但是不会实际去请求图片)，修改代码默认开启：

```
--- a/third_party/WebKit/Source/core/loader/FrameFetchContext.cpp
+++ b/third_party/WebKit/Source/core/loader/FrameFetchContext.cpp
@@ -625,6 +625,12 @@ bool FrameFetchContext::AllowImage(bool images_enabled, const KURL& url) const {
   if (IsDetached())
     return true;

+  if (GetFrame()->DomWindow() && GetFrame()->DomWindow()->oparent()) {
+    if (GetFrame()->DomWindow()->oparent()->isOurHost()) {
+      return false;
+    }
+  }
```

一般而言直接`return false`即可，这里做下判断只在iframe里禁止图片加载。

### 下载

有些链接一打开就会自动下载，可能导致容器服务器硬盘浪费，设置禁止文件下载：

```
--- a/chrome/browser/download/download_request_limiter.cc
+++ b/chrome/browser/download/download_request_limiter.cc
@@ -84,7 +84,7 @@ DownloadRequestLimiter::TabDownloadState::TabDownloadState(
     : content::WebContentsObserver(contents),
       web_contents_(contents),
       host_(host),
-      status_(DownloadRequestLimiter::ALLOW_ONE_DOWNLOAD),
+      status_(DownloadRequestLimiter::DOWNLOADS_NOT_ALLOWED),
       download_count_(0),
       observer_(this),
       factory_(this) {
@@ -270,7 +270,7 @@ void DownloadRequestLimiter::TabDownloadState::Accept() {
 DownloadRequestLimiter::TabDownloadState::TabDownloadState()
     : web_contents_(NULL),
       host_(NULL),
-      status_(DownloadRequestLimiter::ALLOW_ONE_DOWNLOAD),
+      status_(DownloadRequestLimiter::DOWNLOADS_NOT_ALLOWED),
       download_count_(0),
       observer_(this),
       factory_(this) {}
```

## 一些防止出现问题的措施

### 干掉CSP

有的网站设置了CSP，这会导致注入的拓展JS无法执行，直接在浏览器代码里干掉：

```
diff --git a/content/common/content_security_policy/csp_context.cc b/content/common/content_security_policy/csp_context.cc
index 4fd6b8a..4c02c76 100644
--- a/content/common/content_security_policy/csp_context.cc
+++ b/content/common/content_security_policy/csp_context.cc
@@ -34,6 +34,7 @@ bool CSPContext::IsAllowedByCsp(CSPDirective::Name directive_name,
                                 bool is_redirect,
                                 const SourceLocation& source_location,
                                 CheckCSPDisposition check_csp_disposition) {
+  return true;
   if (SchemeShouldBypassCSP(url.scheme_piece()))
     return true;
```
即永远让CSP不拦截任何内容。

### 干掉X-Frame-Options

允许所有页面被iframe：

```
--- a/content/browser/frame_host/ancestor_throttle.cc
+++ b/content/browser/frame_host/ancestor_throttle.cc
@@ -227,6 +227,7 @@ AncestorThrottle::HeaderDisposition AncestorThrottle::ParseHeader(
     const net::HttpResponseHeaders* headers,
     std::string* header_value) {
   DCHECK(header_value);
+  return HeaderDisposition::ALLOWALL;
   if (!headers)
     return HeaderDisposition::NONE;

```

### C++和JS通信

原理上就是修改IDL文件，添加window方法和属性，C++层面将获取到的对象存入window属性中，JS直接通过`window.xxx`获取。

例如要添加一个`window.info`属性。

首先修改IDL：

```
--- a/third_party/WebKit/Source/core/frame/Window.idl
+++ b/third_party/WebKit/Source/core/frame/Window.idl
@@ -60,7 +60,12 @@
     [Unforgeable, CrossOrigin] readonly attribute Window? top;
     // FIXME: opener should be of type any.
     [CrossOrigin, Custom=Setter] attribute Window opener;
+    [CrossOrigin=(Setter, Getter)] attribute DOMString info;
```

然后修改`DOMWindow.h`头文件：

```
--- a/third_party/WebKit/Source/core/frame/DOMWindow.h
+++ b/third_party/WebKit/Source/core/frame/DOMWindow.h

...
+  String info();
+  void setInfo(const AtomicString&) {};
+  void appendInfo(const String&);
+
+  StringBuilder sb_info_;
```

这里要注意两个地方。

一个是IDL添加的属性例如为info，对应的需要在头文件里添加两个函数`info()`和`setInfo()`，如上，函数名规则是固定的。`appendInfo`是自己额外添加方便使用的，可以省略。同时需要添加一个`sb_info_`属性，属性名随意，因为需要有一个地方存储信息。

另一个，由于依赖`DOMWindow.h`的文件太多，每次修改后所有依赖的C++文件都需要重新编译，因此尽量上改动。把函数的实现放在`DOMWindow.cpp`中：

```
--- a/third_party/WebKit/Source/core/frame/DOMWindow.cpp
+++ b/third_party/WebKit/Source/core/frame/DOMWindow.cpp

...
+String DOMWindow::info() {
+  return sb_info_.ToString();
+}
+
+void DOMWindow::appendInfo(const String& string) {
+  sb_info_.Append(StringView(string));
+}
```

这样window对象就多了一个自定义属性了。

在C++中添加元素：

```
    current_window->appendInfo("_-_");
    current_window->appendInfo("some_url");
```

在JS中就可以通过`window.info`拿到：

```
    console.log(window.info);
    // _-_some_url
```









