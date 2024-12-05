---
title: 在 Android Fragment 中集成 React Native
date: 2024-09-09 13:27:38
tags: 
  - Android
  - React Native
permalink: integrate-react-native-to-android-fragment/
---

这里想介绍的是如果我们想在一个已有的项目里独立的集成 React Native，可能我只需要一个特定的简单页面通过 React Native 实现动态化，并不想像文档那样接入 React Native 的 gradle 插件来实现。

先增加 ReactNative 的依赖到 app 模块的 build.gradle

```kotlin
dependencies {
   implementation("com.facebook.react:react-android:0.75.2")
   implementation("com.facebook.react:hermes-android:0.75.2")
}
```

Sync 项目之后可以实现 ReactNative 的入口 Fragment 文件。

```kotlin
class ReactNativeFragment : Fragment() {

    private lateinit var rootView: ReactRootView
    private lateinit var reactInstanceManager: ReactInstanceManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 初始化 RN 的 so 库
        SoLoader.init(requireContext(), false)

        rootView = ReactRootView(requireContext())
        reactInstanceManager = ReactInstanceManager.builder()
            // 传入 Application 实例
            .setApplication(globalApp)
            .setCurrentActivity(requireActivity())
            // 传入 asset 文件中 RN bundle 的文件名，也可以传入本地文件
            .setBundleAssetName("index.android.bundle")
            .setUseDeveloperSupport(false)
            // 这里可以通过传入自定义的 package 来支持 RN 原生模块和原生 View 组件
            .addPackages(listOf(MainReactPackage()))
            .setInitialLifecycleState(LifecycleState.RESUMED)
            .setJSMainModulePath("index")
            .setDefaultHardwareBackBtnHandler {
                Log.d(TAG, "onCreate: defaultHardwareBackHandler handle")
            }
            .build()

        // 启动 RN 应用
        rootView.startReactApplication(
            reactInstanceManager,
            // 传入 RN 中通过 AppRegistry.registerComponent() 注册的组件名
            "react_native_cli_init",
            // 在这里通过 Bundle 给 RN 组件传递参数，RN 组件会通过 props 接收参数
            arguments
        )
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, 
        savedInstanceState: Bundle?
    ): View = rootView // 返回创建的 ReactRootView 实例
}
```

这样就实现了基本的 React Native 组件嵌入 Android Fragment 中。