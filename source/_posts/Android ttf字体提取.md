---
title: Android TTF格式字体提取(避免添加整个字体包)

date: 2017-06-03 11:23:00

tags:

- Android
- TTF
- font


---
# ttf字体提取  

**Android或则其它地方需要用到一些特殊字体，然后字体库都是几兆的大小，全部导入会导致包的大小激增，如何解决这个问题？**


* 首先，安装sfnttool字体提取工具并集成java环境(废话)
* 命令行到sfnttool目录下，输入java -jar sfnttool.jar  -s '字符提取' test.ttf test_exact.ttf  
* 字符文件就被提取到同目录下的test_exact.ttf中


# ttf在Android Studio的应用
* Android studio project的assets目录下有一个fonts目录,没有的话就新建一个fonts目录并且将ttf文件导入到fonts目录下。
* 可以开始在代码中使用，如下所示：

  **TextView tv = getTextView();  
  Typeface typeface = Typeface.createFromAsset(getAssets(),"fonts/bluewhale.ttf");  
  tv.setTypeface(tupeface);**