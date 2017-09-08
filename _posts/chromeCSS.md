
CSS解析在 https://cs.chromium.org/chromium/src/third_party/WebKit/Source/core/css/parser/CSSParserImpl.cpp?type=cs&l=405

里面的type在 make_css_tokenizer_codepoints:

https://cs.chromium.org/chromium/src/third_party/WebKit/Source/build/scripts/make_css_tokenizer_codepoints.py?q=make_css_tokenizer_codepoints&dr

注意这里 \" \' 都是quote，可以混用。

空格包括 '\n\r\t\f '

