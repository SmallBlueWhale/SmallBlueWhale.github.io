---
title: Android环境分离(分离测试和正式环境)
date: 2017-04-11 11:23:00

tags:

- Android
- gradle


---
# Android环境分离  

**昨天项目写了个bug,这里原因其实是测试环境以及正式环境总是变换导致各种第三方id以及url在测试中忘记改回来，然后我觉得人总是没有机器智能的，因此代码的控制对我们程序员还是很有必要的，在这里记录一下Android的环境分离，下一次就不会因为忘记改回来出现bug了。**



### Build Variants

#### Product flavors

1.可以用视图去控制(和你在gradle中使用代码控制是一样的)

首先第一种方式，用flavors去控制环境分离