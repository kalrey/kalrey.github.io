
---
layout: post
title: "play video inline"
description: ""
category:
tags: []
---
{% include JB/setup %}

如何在ios中将一段在线视频内置播放而不调用本地播放器

我们可以利用h5的video标签


####In iOS 8 & iOS 9
>Html:

```
<video src="xxx.mp4" webkit-playsinline autoplay/>
```


>Objective-C:

iOS10 以前，allowsInlineMediaPlayback在iphone上默认为NO,需要设置为YES

```
    //允许內联播放
	webView.allowsInlineMediaPlayback = YES;
	//无需用户手动操作(配合autoplay标签的)
	webView.mediaPlaybackRequiresUserAction = NO;
```

####In iOS 10+

>Html:
注意,iOS10以后，webkit-playsinline变更为playsinline

```
<video src="xxx.mp4" playsinline autoplay/>


>Objective-C:

iOS10+以后，allowsInlineMediaPlayback默认为YES,我们仅需设置自动播放部分

```
	//无需用户手动操作(配合autoplay标签的)
	webView.mediaPlaybackRequiresUserAction = NO;
```


以上信息亲自验证!
