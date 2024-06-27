---
title: HarmonyOS 鸿蒙上使用 Compose imePadding 无效的问题
date: 2024-06-27 09:54:53
tags:
---

# 先说结论

Manifest 中 Activity 的 `android:windowSoftInputMode` 标记避免使用 adjustNothing，而是使用 adjustResize 等其他 adjust 值。

--- 

使用 Compose 构建的登录页面在鸿蒙系统上遇到了给 Composable 组件设置 imePadding 无效的问题，在网络上没有找到相关的博客，大家都在关注使用 KMP 构建纯血鸿蒙的方案。

想到了 (处理输入法可见性)[https://developer.android.com/develop/ui/views/touch-and-input/keyboard-input/visibility?hl=zh-cn#Respond]，才发现我项目中设置的是 `adjustNothing`，改成 `adjustResize` 就可以了。