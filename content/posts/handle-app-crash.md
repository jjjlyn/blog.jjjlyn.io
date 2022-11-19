---
title: "앱 크래시 대응 방법"
date: 2022-11-18T13:39:56+09:00
draft: false
tags:
- Android
---

```kt
@HiltAndroidApp
class CustomApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        setExceptionHandler()
    }

    private fun setExceptionHandler() = Thread.setDefaultUncaughtExceptionHandler(
        CustomErrorHandler.init {
            application(this@CustomApplication)
            defaultHandler(Thread.getDefaultUncaughtExceptionHandler()!!)
        }
    )
}
```

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
