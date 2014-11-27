title: iOS国际化过程中的一些细节
date: 2014-11-26 20:03:34
categories: iOS
tags: [iOS, Xcode]
---

今天下午研究了一下iOS app的国际化，发现里面还有很多的细节是需要注意的，所以在此总结一下。

<!-- more -->

##1. 基础

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


Base internationalization 将面向用户的字符串从 .storyboard和 .xib文件中分离开，这降低了对.storyboard和 .xib文件进行国际化的难度。一个app只有一组.storyboard和 .xib文件，其中字符串是development language的。当我们导出进行国际化的时候，development language 字符串会做为源翻译为多种语言，当导入国际化文件的时候，Xcode会为每一个.storyboard和 .xib文件生成具体语言的字符串。

面向用户的字符串也有可能出现在代码中，这时我们就要使用宏`NSLocalizedString`，当我们导出需要做国际化的时候，Xcode会找到代码中找到这些宏，并把其导出为一个文件，方便做国际化。当导入国际化文件时，Xcode会自动的在工程中添加string文件。

iOS中获取当前语言的方法：
```objective-c
NSString *languageID = [[NSBundle mainBundle] preferredLocalizations].firstObject;
```

##2. 通过本地设置格式化data
通过下面两个方法可以返回当前用户设置的Locale：
```objective-c
NSLocale *userLocale = [NSLocale currentLocale];
NSLocale *userLocale = [NSLocale autoupdatingCurrentLocale];
```
其中currentLocale方法保证返回对象的属性值不会改变，而autoupdatingCurrentLocale返回对象的属性可能改变，当用户改变region设置的时候。

例如，我们可以通过NSLocale获取货币符号，等等。。
```objective-c
NSString *currencySymbol = [[NSLocale currentLocale] objectForKey:NSLocaleCurrencySymbol];   // "￥"
```

获取当前系统语言的可视化字符串通过如下方法：
```objective-c
    NSString *languageID = [[NSBundle mainBundle] preferredLocalizations].firstObject;
    NSLocale *locale = [NSLocale localeWithLocaleIdentifier:languageID];
    NSString *localizedString = [locale displayNameForKey:NSLocaleIdentifier value:languageID];
```
当系统语言为简体中文的时候，languageID的结果为`zh-Hans`，localizedString的结果为`中文（简体中文）`。

在处理NSDate的时候，我们一般使用NSDateFormatter来进行国际化。
对于NSDateFormatter我们可以使用系统预设定的格式进行显示，使用方法`+ (NSString *)localizedStringFromDate:(NSDate *)date dateStyle:(NSDateFormatterStyle)dstyle timeStyle:(NSDateFormatterStyle)tstyle`，系统提供了四种：
```objective-c
typedef NS_ENUM(NSUInteger, NSDateFormatterStyle) {    // date and time format styles
    NSDateFormatterNoStyle = kCFDateFormatterNoStyle,
    NSDateFormatterShortStyle = kCFDateFormatterShortStyle,
    NSDateFormatterMediumStyle = kCFDateFormatterMediumStyle,
    NSDateFormatterLongStyle = kCFDateFormatterLongStyle,
    NSDateFormatterFullStyle = kCFDateFormatterFullStyle
};
```
在中文的locale下，下面是对应的显示：
```objective-c
    NSString *localizedDateTime = [NSDateFormatter localizedStringFromDate:date dateStyle:NSDateFormatterFullStyle timeStyle:NSDateFormatterFullStyle];
    //2014年11月27日 星期四 中国标准时间13:19:21
    localizedDateTime = [NSDateFormatter localizedStringFromDate:date dateStyle:NSDateFormatterLongStyle timeStyle:NSDateFormatterLongStyle];
    //2014年11月27日 GMT+813:19:21
    localizedDateTime = [NSDateFormatter localizedStringFromDate:date dateStyle:NSDateFormatterMediumStyle timeStyle:NSDateFormatterMediumStyle];
    //2014年11月27日 13:19:21
    localizedDateTime = [NSDateFormatter localizedStringFromDate:date dateStyle:NSDateFormatterShortStyle timeStyle:NSDateFormatterShortStyle];
    //14/11/27 13:19
```
当为`NSDateFormatterNoStyle`的时候，格式化结果为空字符串。

当预设定的格式不满足需求的时候，我们可以自定义格式化模板，使用方法`+ (NSString *)dateFormatFromTemplate:(NSString *)tmplate options:(NSUInteger)opts locale:(NSLocale *)locale`。

如下：
```objective-c
    NSDateFormatter *dateFormatter = [NSDateFormatter new];
    NSString *localeFormatString = [NSDateFormatter dateFormatFromTemplate:@"dMMM" options:0 locale:dateFormatter.locale];
    dateFormatter.dateFormat = localeFormatString;
    NSString *localizedString = [dateFormatter stringFromDate:[NSDate date]];
```
在简体中文locale下，结果为`11月27日`，在English (United States) 下，结果为`Nov 27`。

另外一个用户可以设置的为calendar，使用下面方法获取当前用户的设置：
```objective-c
NSCalendar *currentCalendar = [NSCalendar currentCalendar];
```
使用`NSDateComponents`对象来访问calendar中的具体部分：
```objective-c
    NSDateComponents *components = [[NSCalendar currentCalendar]
                                    components:NSCalendarUnitYear | NSCalendarUnitMonth | NSCalendarUnitDay | NSCalendarUnitEra
                                    fromDate:[NSDate date]];
    
    NSInteger day = [components day];
    NSInteger month = [components month];
    NSInteger year = [components year];
    NSInteger era = [components era];
```
对于公历，era是没有作用的。
