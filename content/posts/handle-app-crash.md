---
title: "앱 크래시(비정상적 종료) 대응 방법"
date: 2022-11-18T13:39:56+09:00
draft: false
tags:
- Android
---

어플리케이션이 갑자기 죽었을 때 아무 예고없이 꺼진다면 사용자 경험을 크게 해칩니다. 프로그램이 종료되는 이유는 다양한데, 주로 누락된 예외(Runtime Exception)처리에 의해 발생합니다.

앱의 비정상적 종료는 개발 버전과 프로덕션 버전으로 나누어서 대응할 수 있습니다. 개발 버전은 개발 팀이 알기 쉽게 예외가 발생한 즉시 로그를 화면에 나타내고, 프로덕션 버전은 실제 사용자가 사용하기 때문에 특정 오류 발생 화면으로 전환시켜 사용자 경험을 훼손시키지 않는 동시에, 예외 로그를 실시간으로 알림받도록 합니다.

대응 계획은 아래와 같습니다.

### 개발 버전 대응

- 예외가 발생한 경우 그 즉시 개발 버전의 오류 발생 화면으로 전환시킵니다.

    ![디버그 버전 앱 크래시](/images/android/app-crash-debug.jpeg)

### 프로덕션 버전 대응

- 예외가 발생한 경우 그 즉시 프로덕션 버전의 오류 발생 화면으로 전환시킵니다.

    ![프로덕션 버전 앱 크래시](/images/android/app-crash-production.png)
- 추가로 오류 내역을 Firebase Crashlytics로 전송합니다.

### 클라이언트 별 버전 분리 방법

같은 어플리케이션을 빌드하지만, 클래스 혹은 리소스를 구별하고 싶을 때 Flavors를 사용할 수 있습니다. Flavors는 같은 앱의 여러가지 버전 빌드를 지원합니다. 예를 들어 유/무료 버전을 구분하여 빌드할 수 있습니다.

Flavors를 지정하려면 `build.gradle` 파일 `android` 블록 내부에 `productFlavors` 블록을 삽입해야 합니다. 아래와 같은 방식입니다.

```gradle
android {
    productFlavors {

    }
}
```

앱 전체 `build.gradle` 파일에 Flavors와 함께 빌드될 apk 파일명을 추가합니다. 여기서는 `development`와 `production`이라는 Flavors를 추가하겠습니다.

```gradle
allprojects {
    android {
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
}
```

앱을 멀티모듈로 구성했기 때문에 오류 화면을 담당하는 CustomErrorActivity는 공통 자원이라고 판단하여 `:core` 모듈에 위치시켰습니다.

Flavors로 `development`, `production`을 추가했으므로 각 폴더(`/development`, `/production`)을 `:core` 모듈 안에 생성합니다. 생성된 디렉토리에 버전 별 오류 화면(CustomErrorActivity)과 레이아웃(`R.layout.activity_custom_error`)을 추가합니다.

![Flavors 설정](/images/android/flavor.png)

`Application`에서 커스텀 익셉션 핸들러를 `DefaultUncaughtExceptionHandler`로 설정합니다.

```kt
package app.asdf.android

class CustomApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        setExceptionHandler()
    }

    private fun setExceptionHandler() {
        Thread.setDefaultUncaughtExceptionHandler(
            // 앞으로 예외 발생 시 커스텀 핸들러가 작동합니다.
            CustomErrorHandler.init {
                application(this@CustomApplication)
                defaultHandler(Thread.getDefaultUncaughtExceptionHandler()!!)
            }
        )
    }
}
```

커스텀 에러 핸들러를 구현합니다. Flavor에 종속되지 않는 공통 소스이므로 `:core` 모듈의 `/main/java/{asdf}/core` 디렉토리에 배치합니다. 오류 화면을 `finish()`하면 가장 최근 실행된 Activity로 돌아가야 하기 때문에, 오류 화면으로 전환하기 바로 이전 Activity를 항상 참조해야 합니다. 따라서 CustomErrorHandler에서 Activity의 생명주기 변화에 따른 콜백을 등록합니다.

```kt
package app.asdf.core.error

class CustomErrorHandler private constructor(application: Application, private val defaultExceptionHandler: Thread.UncaughtExceptionHandler) : Thread.UncaughtExceptionHandler {

    private var lastActivity: WeakReference<Activity>? = null
    private var activityCount = 0

    init {
        application.registerActivityLifecycleCallbacks(
            object : Application.ActivityLifecycleCallbacks {
                override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
                    // 오류 화면이 실행된 경우 return (무시)
                    if (isErrorActivity(activity)) {
                        return
                    }
                    lastActivity = WeakReference(activity)
                }

                override fun onActivityStarted(activity: Activity) {
                    // 오류 화면이 실행된 경우 return (무시)
                    if (isErrorActivity(activity)) {
                        return
                    }
                    activityCount++
                    lastActivity = WeakReference(activity)
                }

                override fun onActivityResumed(activity: Activity) = Unit
                override fun onActivityPaused(activity: Activity) = Unit

                override fun onActivityStopped(activity: Activity) {
                    // 오류 화면이 실행된 경우 return (무시)
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

개발 버전 오류 화면 입니다. `:core`모듈의 `/development/java/app/{asdf}/core/error` 경로에 배치합니다.

```kt
package app.asdf.core.error

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
        // 예외 로그를 화면 상에 표시합니다.
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

프로덕션 버전 오류 화면 입니다. `:core`모듈의 `/production/java/app/{asdf}/core/error` 경로에 배치합니다.

```kt
package app.asdf.core.error

class CustomErrorActivity : AppCompatActivity() {

    private val binding: ErrorCustomActivityBinding by viewBindings(ErrorCustomActivityBinding::inflate)

    private val lastActivityIntent by lazy { intent.getParcelableExtra<Intent>(EXTRA_INTENT) }
    private val exception: Throwable by lazy { intent.getSerializableExtra(EXTRA_ERROR) as Throwable }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        
        // crashlytics에 예외 로그를 전송하여 개발 팀에서 실시간으로 대응할 수 있도록 합니다.
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

### 정리

- 직접 구현한 CustomErrorHandler를 DefaultUncaughtExceptionHandler로 설정하여 Runtime Exception 발생 시 CustomErrorHandler가 호출되도록 합니다.
- `build.gradle`의 Flavors 설정을 통해 두 가지 버전(개발/운영)을 추가하고, 이에 따라 각각 다른 클래스와 리소스를 제공하여 사용자에게 특정 버전에 맞는 오류 화면을 보여줄 수 있습니다.