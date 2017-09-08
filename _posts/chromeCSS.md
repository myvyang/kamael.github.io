
CSS解析在 https://cs.chromium.org/chromium/src/third_party/WebKit/Source/core/css/parser/CSSParserImpl.cpp?type=cs&l=405

里面的type在 make_css_tokenizer_codepoints:

https://cs.chromium.org/chromium/src/third_party/WebKit/Source/build/scripts/make_css_tokenizer_codepoints.py?q=make_css_tokenizer_codepoints&dr

注意这里 \" \' 都是quote，可以混用。

空格包括 '\n\r\t\f '。

CSS词法解析是无状态的，只是一个个读取字符，生成token，然后把token流组成合适的CSS规则。

fuzz点：

```
<head>
<style>

.aaa {
  background-color : black;
  background-url: "/aaa</style/";
}
</style>
</head>

<body>

```

`<style`后面是否浏览器会闭合，而过滤器不会？如果是这样，会导致解析异常。
