---
title: "액세스 토큰을 어떻게 단 한번만 갱신할 수 있을까?"
date: 2022-11-18T13:39:56+09:00
draft: false
categories:
- Android
tags:
- Android
---

```kt
@Singleton
class AuthInterceptor @Inject constructor(
    context: Context,
    private val authDataStore: AuthDataStore,
    @UDIDInterceptor private val udidInterceptor: Interceptor,
    @StethoInterceptor private val stethoInterceptor: Interceptor
) : Interceptor {

    private val gson = GsonBuilder().setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSXXX").create()
    private val mutex = Mutex()
    private var limitCnt = 0

    private val okHttpClient = OkHttpClient.Builder()
        .cache(Cache(context.cacheDir, 1 * 1024 * 1024)) // 1 MB
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
            else HttpLoggingInterceptor.Level.NONE
        })
        .followRedirects(false)
        .addInterceptor(OkHttpInterceptors.createOkHttpInterceptor())
        .addNetworkInterceptor(OkHttpInterceptors.createOkHttpNetworkInterceptor())
        .addNetworkInterceptor(udidInterceptor)
        .addNetworkInterceptor(stethoInterceptor)
        .build()

    private val retrofit = Retrofit.Builder()
        .baseUrl(apiEndPoint())
        .addConverterFactory(NullOrEmptyConverterFactory())
        .addConverterFactory(GsonConverterFactory.create(gson))
        .addConverterFactory(EnumConverterFactory())
        .addCallAdapterFactory(NetworkResponseAdapterFactory())
        .client(okHttpClient)
        .build()

    private fun Interceptor.Chain.proceedWithToken(request: Request, token: String): Response {
        return if (limitCnt > 100) {
            limitCnt = 0
            throw IllegalArgumentException()
        } else {
            request
                .newBuilder()
                .removeHeader("Authorization")
                .addHeader("Authorization", "Bearer $token")
                .build()
                .let(::proceed)
        }
    }

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().also {
            Timber.d("[1] $it")
        }
        // no coroutine can execute on this thread (main) until the blocking call is finished
        val token = runBlocking {
            authDataStore.getTokenInfo().first()
        }.also { Timber.d("[2] $request $it") }
        val response = chain.proceedWithToken(request, token.accessToken)
        if (response.code != StatusCode.AUTHENTICATE_FAILED.code) {
            limitCnt = 0
            return response
        }
        limitCnt++
        Timber.d("[3] $request")
        val newToken: String? = runBlocking {
            mutex.withLock {
                val tokenResult = authDataStore.getTokenInfo().first().also { Timber.d("[4] $request") }
                val maybeUpdatedToken = tokenResult.accessToken

                when {
                    tokenResult.refreshToken.isEmpty() -> null
                    maybeUpdatedToken != token.accessToken -> maybeUpdatedToken.also {
                        Timber.d("[5-1] $request")
                    }
                    else -> {
                        Timber.d("[5-2] $request")

                        val authService = retrofit.create(AuthService::class.java)
                        val refreshTokenResponse =
                            authService.fetchDefaultTokenRes(
                                DefaultToken(
                                    key = tokenResult.refreshToken,
                                    provider = "REFRESH-TOKEN"
                                )
                            ).also { Timber.d("[6] $request") }
                        val code = refreshTokenResponse.code()
                        if (code == StatusCode.SUCCESS.code) {
                            refreshTokenResponse.body().also { tr ->
                                if (tr != null) {
                                    authDataStore.saveTokenInfo(tr.copy(provider = tokenResult.provider))
                                }
                            }?.accessToken
                        } else if (code == StatusCode.AUTHENTICATE_FAILED.code) {
                            null
                        } else if (code == StatusCode.FORBIDDEN.code){
                            authDataStore.popUpBadGateWayDialog(Event(Unit))
                            null
                        } else {
                            null
                        }
                    }
                }
            }
        }
        return if (newToken != null) {
            chain.proceedWithToken(request, newToken)
        }
        else {
            response
        }
    }
}
```