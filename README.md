# flutter灵动岛效果实现

form https://juejin.cn/post/7154420798059446309#heading-3

在`android/app/src/main/AndroidManifest.xml`文件有这一段：用于创建后台服务
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.karl.open.desktop_app">
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
    <!-- 启动后台服务 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE " />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <application
        android:icon="@mipmap/ic_launcher"
        android:label="desktop_app">

        <service
            android:name=".WindowsService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="OpenIsland" />
            </intent-filter>
        </service>
```
Android小组件与Windows可是大有不同。由于Google基于安全的限制，Android应用必须是全屏且不允许穿透点击，因此Android的小组件一般都是依附于悬浮窗来开发的，即windows_manager。<br>
Flutter只是一个UI框架，自然也不能脱离Android本身的机制，因此我们需要在原生层创建一个悬浮窗，然后创建一个Flutter engine来吸附Flutter的UI<br>
## 注意
  通过服务唤起悬浮窗，Android要求必须是系统应用，因此大家在使用的时候还需要配置下系统签名；
Flutter engine必须使用FlutterEngineGroup进行托管，否则静置一段时间后，engine就会被系统回收！！！
- 唤起悬浮窗组件
直接通过adb指令唤起即可
```shell
adb shell am start-foreground-service -n com.karl.open.desktop_app/com.karl.open.desktop_app.WindowsService
```

- 使用 kt 语言绘制启动效果
```kotlin
package com.karl.open.desktop_app
import android.annotation.SuppressLint
import android.app.Service
import android.content.Intent
import android.graphics.PixelFormat
import android.os.IBinder
import android.util.DisplayMetrics
import android.view.LayoutInflater
import android.view.MotionEvent
import android.view.ViewGroup
import android.view.WindowManager
import android.widget.FrameLayout
import com.karl.open.desktop_app.utils.Utils
import io.flutter.embedding.android.FlutterSurfaceView
import io.flutter.embedding.android.FlutterView
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.embedding.engine.FlutterEngineGroup
import io.flutter.embedding.engine.dart.DartExecutor
import io.flutter.view.FlutterMain.findAppBundlePath

class WindowsService : Service() {
    // Flutter引擎组，可以自动管理引擎的生命周期
    private lateinit var engineGroup: FlutterEngineGroup

    private lateinit var engine: FlutterEngine

    private lateinit var flutterView: FlutterView
    private lateinit var windowManager: WindowManager

    private val metrics = DisplayMetrics()
    private lateinit var inflater: LayoutInflater

    @SuppressLint("InflateParams")
    private lateinit var rootView: ViewGroup

    private lateinit var layoutParams: WindowManager.LayoutParams

    override fun onCreate() {
        super.onCreate()
        layoutParams = WindowManager.LayoutParams(
            Utils.dip2px(this, 168.toFloat()),
            Utils.dip2px(this, 30.toFloat()),
            WindowManager.LayoutParams.TYPE_SYSTEM_ALERT,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT
        )
        
        // 初始化变量
        windowManager = this.getSystemService(Service.WINDOW_SERVICE) as WindowManager
        inflater =
            this.getSystemService(Service.LAYOUT_INFLATER_SERVICE) as LayoutInflater
        rootView = inflater.inflate(R.layout.floating, null, false) as ViewGroup
        engineGroup = FlutterEngineGroup(this)
        
        // 创建Flutter Engine
        val dartEntrypoint = DartExecutor.DartEntrypoint(findAppBundlePath(), "main")
        val option =
            FlutterEngineGroup.Options(this).setDartEntrypoint(dartEntrypoint)
        engine = engineGroup.createAndRunEngine(option)
        
        // 设置悬浮窗的位置
        @Suppress("Deprecation")
        windowManager.defaultDisplay.getMetrics(metrics)
        setPosition()
        @Suppress("ClickableViewAccessibility")
        rootView.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    layoutParams.flags =
                        layoutParams.flags or WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    windowManager.updateViewLayout(rootView, layoutParams)
                    true
                }
                else -> false
            }
        }
        
        engine.lifecycleChannel.appIsResumed()

        // 为悬浮窗加入布局
        rootView.findViewById<FrameLayout>(R.id.floating_window)
            .addView(
                flutterView,
                ViewGroup.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.MATCH_PARENT
                )
            )
        windowManager.updateViewLayout(rootView, layoutParams)
    }
    

    private fun setPosition() {
        // 设置位置
        val screenWidth = metrics.widthPixels
        val screenHeight = metrics.heightPixels
        layoutParams.x = (screenWidth - layoutParams.width) / 2
        layoutParams.y = (screenHeight - layoutParams.height) / 2

        windowManager.addView(rootView, layoutParams)
        flutterView = FlutterView(inflater.context, FlutterSurfaceView(inflater.context, true))
        flutterView.attachToFlutterEngine(engine)
    }
}
```
