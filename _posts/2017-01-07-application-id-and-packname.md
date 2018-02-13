---
layout: post
title: application ID 与 PackName
category: Android
tags: Android
---

近日为英语✓添加7.1的 App shortCut 功能，实现静态 ShortCut 只需要改动两个地方。一是在 AndroidManifest.xml 中为目标Activity添加
````
<meta-data android:name="android.app.shortcuts"
    android:resource="@xml/shortcuts" />
````
二就是配置 shortcuts.xml
````
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
  <shortcut
    android:shortcutId="compose"
    android:enabled="true"
    android:icon="@drawable/compose_icon"
    android:shortcutShortLabel="@string/compose_shortcut_short_label1"
    android:shortcutLongLabel="@string/compose_shortcut_long_label1"
    android:shortcutDisabledMessage="@string/compose_disabled_message1">
    <intent
      android:action="android.intent.action.VIEW"
      android:targetPackage="com.example.myapplication"
      android:targetClass="com.example.myapplication.ComposeActivity" />
    <!-- If your shortcut is associated with multiple intents, include them
         here. The last intent in the list determines what the user sees when
         they launch this shortcut. -->
    <categories android:name="android.shortcut.conversation" />
  </shortcut>
  <!-- Specify more shortcuts here. -->
</shortcuts>
````
这样，shortcut 就已经完成了。

但是，当我配置完成测试时 Nova启动器直接崩溃，pixel启动器显示“App could not install”。问题出在 `targetPackage`属性。这里的 `targetPackage` 指在build.gradle 中的 `applicationId` 字段，而不是 AndroidManifest 中的 packname。

packname 是 Java 语言所用的，在代码中作为命名空间。android 中使用 applicationId 作为一个应用程序的唯一标识，通常说的包名事实上指的是 applicationId。applicationId 会覆盖 packname ，官网说明如下：
> Although you may have a different name for the manifest `package` and the Gradle `applicationId`, the build tools copy the application ID into your APK's final manifest file at the end of the build. So if you inspect your `AndroidManifest.xml` file after a build, don't be surprised that the `package` attribute has changed.

当初建项目时没有留意，packname 使用了 applicationId 名称的大写，复制 packname 到 shortcut.xml 中的 targetPackage 会导致启动器找不到包。
		