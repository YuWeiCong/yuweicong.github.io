---
layout:     post
title:      Android Webview WebSettings
date:       2020-01-03
author:     Hope
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Chromium
    - Android Webview
---
# WebSettings

以下是Android Webview的WebSettings的UML类图

![drawio](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/WebViewWebSettings.jpg)

### android.webkit.WebSettings
android.webkit.WebSettings是Android Webview对外暴露的webview相关属性的类。包含如下几大类的设置：

- InitialPageScale
- WebPreferences(WebkitPreferences)
- ScroolAndScaleState
- RendererPreferences
- OffecreenPreRaster
- WillSuppressErrorState
- CookiePolicy
- AllowFileAceess

### AwSettings & AwContentBrowserClient & AwSettings.EventHandler
对WebSettings的属性的设置，全都会调用AwSettings。所以AwContentBrowserClient有点像代理模式中的，那个代理者，封装了一个AwSettings。

由于我们可以在任意的线程对WebSettings进行更改，而AwSettings是JNI的Java端接口，这里会存在一个JNI的多线程调用问题。故封装了一个org.chromium.android_webview.AwSettings.EventHandler，确保AwSettings都是在UI线程中调用相关的JNI接口的。，比如在UI线程中调用updateWebkitPreferencesOnUiThreadLocked方法，来通知android_webivew::AwSettings，WebPreferences已经改变了。

### android_webview::AwSettings
android_webview::AwSettings是真正用于更新WebSettings各种状态的地方。所以当更新WebSettings的时候，android_webview::AwSettings的UpdateWebkitPreferencesLocked方法会被调用(JNI)，该方法会调用后content::RenderViewHost的OnWebkitPreferencesChanged方法，来通知content::RenderViewHost来改变content::RenderView的WebPreference。而WebPreference就是真正存储各种状态的地方。

### content::RenderViewHost && content::ContentBrowserClient
content::RenderViewHost会调用ComputeWebPreferences方法来计算当前的WebPreferences。而ComputeWebPreferences方法会调用content::ContentBrowserClient的OverrideWebkitPrefs来设置自定义部分的WebSetting的设置。在 Android Webview中，android_webivew::AwContentBrowserClient会调用android_webview::AwSettings.PopulateWebPreferences来获取自定义的设置部分的WebSetings。完成计算后，content::RenderViewHost会通过IPC消息，调用content::RenderViewImpl的OnUpdateWebPreferences来更新RenderViewImpl的WebPreferences.


上述过程时序图如下：
![drawio](https://raw.githubusercontent.com/YuWeiCong/draw.io/master/chromium/images/WebViewWebSettingsForTiming.jpg)

#### 参考文档：
- [WebView控件之WebSettings详解](https://www.jianshu.com/p/fb6585a7753b)
- [WebSettings官方文档](https://developer.android.com/reference/android/webkit/WebSettings)
