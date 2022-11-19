---
title: "앱 크래시(비정상적 종료) 대응 방법"
date: 2022-11-18T13:39:56+09:00
draft: false
tags:
- Android
---

어플리케이션이 갑자기 죽었을 때 아무 예고없이 꺼진다면 사용자 경험을 크게 해칩니다. 프로그램이 종료되는 이유는 다양한데, 주로 누락된 예외(Runtime Exception)처리에 의해 발생합니다.

앱의 비정상적 종료는 개발(디버그) 버전과 프로덕션 버전으로 나누어서 대응할 수 있습니다.

대응 계획은 아래와 같습니다.

### 개발 버전 대응
- 예외가 발생한 경우 그 즉시 디버그 버전의 오류 발생 화면으로 전환시킵니다.

    ![디버그 버전 앱 크래시](/images/android/app-crash-debug.jpeg)

### 프로덕션 버전 대응
- 예외가 발생한 경우 그 즉시 프로덕션 버전의 오류 발생 화면으로 전환시킵니다.

    ![프로덕션 버전 앱 크래시](/images/android/app-crash-production.png)
- Firebase Crashlytics로 오류 내역을 전송합니다.

### 버전을 분리하는 방법

앱 전체 `build.gradle` 파일에 flavors를 추가합니다.

```gradle
allprojects {
    android {
        flavorDimensions "flavors"
        productFlavors {
            development {
                dimension "flavors"
            }
            production {
                dimension "flavors"
            }
        }
    }
}
```

앱 모듈(`:app`)의 `build.gradle` 파일에도 flavors와 함께 빌드될 apk 파일명을 추가합니다.

```gradle
plugins {
    id 'com.google.firebase.crashlytics'
}

android {
    buildTypes {
        flavorDimensions "flavors"
        productFlavors {
            development {
                dimension "flavors"
                manifestPlaceholders = [ appLabel:"앱 이름(Dev)"]
            }
            production {
                dimension "flavors"
                manifestPlaceholders = [ appLabel:"앱 이름"]
            }
        }
}
```

앱을 멀티모듈로 구성했기 때문에 오류 화면을 담당하는 CustomErrorActivity는 공통 자원이라고 판단하여 `:core` 모듈에 위치시켰습니다.

flavors로 development, production을 추가했으므로 각 폴더(`/development`, `/production`)을 `:core` 모듈 안에 생성합니다. 생성된 디렉토리에 버전 별 오류 화면(CustomErrorActivity)과 레이아웃(`R.layout.activity_custom_error`)을 추가합니다.

![Flavor 설정](/images/android/flavor.png)

`Application`에서 커스텀 익셉션 핸들러를 `DefaultUncaughtExceptionHandler`로 설정합니다.

```kt
class CustomApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        setExceptionHandler()
    }

    private fun setExceptionHandler() = Thread.setDefaultUncaughtExceptionHandler(
        CustomErrorHandler.init { // 앞으로 예외 발생 시 커스텀 에러 핸들러가 작동합니다.
            application(this@CustomApplication)
            defaultHandler(Thread.getDefaultUncaughtExceptionHandler()!!)
        }
    )
}
```

에러 핸들러

```kt
class CustomErrorHandler private constructor(application: Application, private val defaultExceptionHandler: Thread.UncaughtExceptionHandler) : Thread.UncaughtExceptionHandler {

    private var lastActivity: WeakReference<Activity>? = null
    private var activityCount = 0

    init {
        application.registerActivityLifecycleCallbacks(
            object : Application.ActivityLifecycleCallbacks {
                override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
                    if (isErrorActivity(activity)) {
                        return
                    }
                    lastActivity = WeakReference(activity)
                }

                override fun onActivityStarted(activity: Activity) {
                    if (isErrorActivity(activity)) {
                        return
                    }
                    activityCount++
                    lastActivity = WeakReference(activity)
                }

                override fun onActivityResumed(activity: Activity) = Unit
                override fun onActivityPaused(activity: Activity) = Unit

                override fun onActivityStopped(activity: Activity) {
                    if (isErrorActivity(activity)) {
                        return
                    }
                    activityCount--;
                    if (activityCount < 0) {
                        lastActivity = null
                    }
                }

                override fun onActivityDestroyed(activity: Activity) = Unit

                override fun onActivitySaveInstanceState(activity: Activity, savedInstanceState: Bundle)  = Unit
            }
        )
    }

    private fun isErrorActivity(activity: Activity) = activity is CustomErrorActivity

    override fun uncaughtException(thread: Thread, exception: Throwable) {
        lastActivity?.get()?.run {
            val stackTrace = StringWriter()
            exception.printStackTrace(PrintWriter(stackTrace))
            startErrorActivity(this, exception)
        } ?: defaultExceptionHandler.uncaughtException(thread, exception)

        Process.killProcess(Process.myPid())
        exitProcess(-1)
    }

    private fun startErrorActivity(activity: Activity, exception: Throwable) = activity.run {
        val errorActivityIntent = Intent(this, CustomErrorActivity::class.java).apply {
            putExtra(CustomErrorActivity.EXTRA_INTENT, intent)
            putExtra(CustomErrorActivity.EXTRA_ERROR, exception)
        }
        startActivity(errorActivityIntent)
        finish()
    }

    companion object {
        inline fun init(block: Builder.() -> Unit) = Builder().apply(block).build()
    }

    private constructor(builder: Builder) : this(builder.application, builder.defaultErrorHandler)

    class Builder {
        internal lateinit var application: Application
        internal lateinit var defaultErrorHandler: Thread.UncaughtExceptionHandler

        fun application(application: Application) : Builder {
            this.application = application
            return this
        }

        fun defaultHandler(defaultErrorHandler: Thread.UncaughtExceptionHandler) : Builder {
            this.defaultErrorHandler = defaultErrorHandler
            return this
        }

        fun build(): CustomErrorHandler {
            return CustomErrorHandler(this)
        }
    }
}
```

개발 버전 오류 화면 입니다.

```kt
class CustomErrorActivity : AppCompatActivity() {

    private val binding: ErrorCustomActivityBinding by viewBindings(ErrorCustomActivityBinding::inflate)

    private val lastActivityIntent by lazy { intent.getParcelableExtra<Intent>(EXTRA_INTENT) }
    private val exception: Throwable by lazy { intent.getSerializableExtra(EXTRA_ERROR) as Throwable }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        binding.header.toolbar.title = "Error Log :-("
        binding.header.toolbar.setTitleTextColor(ContextCompat.getColor(this, R.color.color_error))

        val stackTrace = StringWriter()
        exception.printStackTrace(PrintWriter(stackTrace))
        binding.textError.text = stackTrace.toString()
        binding.buttonRefresh.setOnClickListener {
            startActivity(lastActivityIntent)
            finish()
        }
        Timber.e(exception)
    }

    companion object {
        const val EXTRA_INTENT = "EXTRA_INTENT"
        const val EXTRA_ERROR = "EXTRA_ERROR"
    }
}
```

프로덕션 버전 오류 화면 입니다.

```kt
class CustomErrorActivity : AppCompatActivity() {

    private val binding: ErrorCustomActivityBinding by viewBindings(ErrorCustomActivityBinding::inflate)

    private val lastActivityIntent by lazy { intent.getParcelableExtra<Intent>(EXTRA_INTENT) }
    private val exception: Throwable by lazy { intent.getSerializableExtra(EXTRA_ERROR) as Throwable }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        
        FirebaseCrashlytics.getInstance().apply {
            recordException(exception)
            sendUnsentReports()
        }

        binding.buttonRefresh.setOnClickListener {
            startActivity(lastActivityIntent)
            finish()
        }
        Timber.e(exception)
    }

    companion object {
        const val EXTRA_INTENT = "EXTRA_INTENT"
        const val EXTRA_ERROR = "EXTRA_ERROR"
    }
}
```
