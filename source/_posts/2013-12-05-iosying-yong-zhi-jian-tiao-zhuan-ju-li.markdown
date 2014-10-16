---
layout: post
title: "iOS应用之间跳转举例"
date: 2013-12-05 11:47
comments: true
categories: iOS
categories: 
---

最近好久没有写文章了。这两天在研究iOS中app之间的跳转，发现网上虽然很多资料，但是都没有详细的举例子，今天打算讲讲。

<!--more-->

##1.两个简单app之间的跳转
(1).首先创建两个工程A和B。
(2).现在我想在B工程中，点击某个按钮跳转到A。则B要知道A定义的URL Types。
(3).下面我们看看A中plist的URL Types是如何定义的：

{% img http://yuzhongleixueren.u.qiniudn.com/2013-12-05-12-04.png 494 114 %}

上面A的plist的大概意思就是：如果外部的程序想要打开我，URL的地址必须要为nav://com.b.nav

(4)现在B中如果点击一个按钮，想要打开A，则应用中应该这么写

```objective-c
- (IBAction)openA:(id)sender
{
	NSString *str = @"nav://com.b.nav?parameter=%@";
    NSURL *url = [NSURL URLWithString:[NSString stringWithFormat:str, @"可以传参数"]];
    [[UIApplication sharedApplication] openURL:url];
}
```

(5)如果在A中又想回到B，则要查看B的plist了。

{% img http://yuzhongleixueren.u.qiniudn.com/2013-12-05-12-51.png 518 114 %}

在A中直接下面代码就回到B了。
```objective-c
- (IBAction)backB:(id)sender
{
	NSString *str = @"test://com.a.test?parameter=%@";
    NSURL *url = [NSURL URLWithString:[NSString stringWithFormat:str, @"可以传参数"]];
    [[UIApplication sharedApplication] openURL:url];
}
```

总结一下，就是在想要跳转，就要知道要跳转到的app中plist URL types的定义。
其实不需要写URL identifier只写URL Schemes也可以同样的跳转
NSString *str = @"nav://parameter=%@"; 就可以打开A了。




