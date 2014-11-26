title: iOS国际化过程中的一些细节
date: 2014-11-26 20:03:34
categories: iOS
tags: [iOS, Xcode]
---

今天下午研究了一下iOS app的国际化，发现里面还有很多的细节是需要注意的，所以在此总结一下。

<!-- more -->

当我们通过Xcode6新建一个工程的时候，工程的目录结构大概是这样子的：

{% img http://ioser.qiniudn.com/2014-11-26-01.png 538 294 %}

Xcode会自动的把 LaunchScreen.xib 和 Main.storyboard 放入到Base.lproj目录中，这是因为在Xcode6中我们新建的工程 Use Base Internationalization 默认是勾选的。

{% img http://ioser.qiniudn.com/2014-11-26-02.png 550 200 %}


当我们添加一种语言之后，(本例子中添加简体中文为例子)，目录结构和工程结构如下：

{% img http://ioser.qiniudn.com/2014-11-26-03.png 550 300 %}

{% img http://ioser.qiniudn.com/2014-11-26-04.png 255 350 %}

Xcode帮我们自动生成了zh-Hans.lproj目录，以及分别本地化xib和storyboard的字符串文件。

如果在代码中需要写入文本的话，我们需要创建Localizable.strings文件，如果创建出来对应的国际化文件，最后结构如下：

{% img http://ioser.qiniudn.com/2014-11-26-04.png 281 238 %}

base的意思就是如果没有相对应的国际化文件，就使用base中提供的。例如当我们把简体中文的Localizable.strings删掉之后，则在简体中文下找不到对应的key了，就会去base找。


另一个需要知道的就是Info.plist中的key CFBundleDevelopmentRegion，此key的意思就是当手机当前的语言环境没有在对应的国际化列表中，就用CFBundleDevelopmentRegion的值对应的语言的国际化文件来显示。默认都是en，但是我们可以修改~


对于iOS来讲，当改变language设置的时候，会重启Springboard，退出所有运行的app，当app 再次运行的时候，其会使用新的language设置。但是region和calendar设置可以在任何时候改变，甚至在app运行的时候。因此，一个国际化的app应该考虑到这些方面。

Region的设置和language的设置是独立的，改变language的设置是不会自动改变region的设置的，如果想要同步，只能自己手动设置。region的设置主要作用于date和time的格式。

Calendar的设置当前只有三种，公历，日本日历和佛教日历。











































