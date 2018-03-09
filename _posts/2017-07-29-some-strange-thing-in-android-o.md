---
layout: post
title: Some Strange Thing in Android O
category: Android
tags: Android
---

### Notification Channel
为兼容各个版本的系统，发布通知通常使用 `NotificationCompat.Builder` 来创建 `Notification`。但对于 Android O 中新增的 Notification Channel，相关 API `setChannel()` 存在问题...以下代码将打印“true”
```java
NotificationChannel mChannel = new NotificationChannel(NOTIFICATION_CHANNEL_ID, NOTIFICATION_CHANNEL_NAME, NotificationManagerCompat.IMPORTANCE_LOW);
mNotificationManager.createNotificationChannel(mChannel);
Notification notification = new NotificationCompat.Builder(context)
        .set...
        .setChannel(NOTIFICATION_CHANNEL_ID);
Log.i(TAG, notification.getChannelId() == null);
```
[官方示例代码](https://github.com/googlesamples/android-NotificationChannels)中使用 `Notification.Builder` 创建通知，这似乎是唯一解决方法...     
相关问题：[https://stackoverflow.com/questions/44489657/android-o-reporting-notification-not-posted-to-channel-but-it-is](https://stackoverflow.com/questions/44489657/android-o-reporting-notification-not-posted-to-channel-but-it-is)

### Settings.ACTION_MANAGE_OVERLAY_PERMISSIONS
对于 SDK >= M 的设备，需要单独申请 `Display over other apps` 权限，通常的作法是用 `Settings.ACTION_MANAGE_OVERLAY_PERMISSION` 构造 `Intent` 启动设置页面，然后判断返回值。但在 Android O 中，`onActivityResult` 方法收到的 `resultCode` 永远为 0 ，即 `RESULT_CANCELED` ，不管用户是否授予权限。在 `onActivityResult` 方法中使用 `Settings.canDrawOverlays` 返回值也为 false。    
相关问题：[https://stackoverflow.com/questions/44828422/android-o-opp3-170518-006-why-is-there-a-delay-when-changing-system-settings](https://stackoverflow.com/questions/44828422/android-o-opp3-170518-006-why-is-there-a-delay-when-changing-system-settings)