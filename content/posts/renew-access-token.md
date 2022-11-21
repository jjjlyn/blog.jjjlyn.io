---
title: "어떻게 액세스 토큰(Access Token)을 단 한번만 갱신할 수 있을까?"
date: 2022-11-20T11:35:56+09:00
draft: false
tags:
- Android
- JWT
---

네트워크 통신을 주로 하는 상용 서비스의 경우 보통 한 화면에서 다수의 API를 호출하게 됩니다. 유명 앱서비스를 패킷 캡쳐로 테스트 해본 적이 있는데 한 화면 실행 시 약 100여 개의 HTTP Request가 들어오더군요.

`OkHttpClient`의 속성 중 하나인 `AuthInterceptor`구현체는 HTTP 관련 특정 에러(e.g. `401 Unauthorized`, `403 Forbidden` 등)에 대한 처리를 담당합니다. 이 `AuthInterceptor`은 HTTP Request가 생성될 때마다 호출됩니다. 앞에서 언급한 모 서비스의 경우 100여 번의 `AuthInterceptor` 로직이 실행될 것입니다.

서비스가 코루틴 블록 내부에서 호출을 하는 경우 약 100여 개의 코루틴이 생성되어 비동기로 돌아갑니다.

앱 서비스는 JWT 토큰으로 인증 관리를 많이 합니다. 비동기로 인증 로직을 실행하면 특정 시점에서 난감한 문제가 발생할 수 있습니다. **바로 액세스 토큰(Access Token)이 만료되어 연쇄적으로 `401 Unauthorized` 오류를 반환하는 경우입니다.** JWT 인증은 액세스 토큰이 만료되었을 때 리프레시 토큰(Refresh Token)을 통해 액세스 토큰을 갱신하는 방법을 취합니다. 단순하게 인증 로직을 짜버리면 100여 개의 코루틴이 비동기로 돌면서 각각 새로운 액세스 토큰을 갱신하게 됩니다. 예를 들어 특정 유저가 화면A에 접근했을 때 공교롭게 인증 기간이 만료되었다고 가정해 봅시다. 화면A는 100개의 API를 호출합니다. 서로 다른 HTTP Request가 HTTP Response를 반환하기 위해서 액세스 토큰을 생성하게 됩니다. 화면 하나 띄우려다가 액세스 토큰이 100번 갱신되는 사태가 발생한 것입니다. 이는 여러 기기에서 동시 접속을 허용할 경우 딱히 문제가 되지 않으나(여전히 쓸데없는 비용이 발생하는 것이다만 작동 상에는 문제가 없음), 허용하지 않는 경우 영원히 인증이 안 되어 Request를 Retry하는 *무한 츠쿠요미...* 현상이 발생합니다. 

동시 접속을 허용하지 않을 때 리프레시 토큰으로 인증 정보를 새롭게 발급받는 경우 액세스 토큰, 리프레시 토큰이 재생성됩니다(기존 액세스 토큰, 리프레시 토큰은 JWT 토큰을 관리하는 Redis DB에서 삭제됩니다). `AuthInterceptor`의 `intercept()` 내 **토큰 갱신 후 저장하는 로직**에 대한 원자성이 보장되지 않으면 의도치 않게 동시에 실행되는 다른 코루틴에서 토큰 정보를 계속 변경할 위험이 있습니다. 이는 역시 Redis DB와 DataStore에도 업데이트 됩니다. 

코루틴 A, B, C가 있다고 가정해 봅시다. 현재 토큰은 만료된 상태라, 갱신 작업을 거쳐 다시 API 요청을 해야 합니다. 동시 접속을 허용하지 않으므로 사용자 계정의 토큰 정보는 단 한개만 유지됩니다. 

- **코루틴 A**에서 토큰 갱신 작업에 들어갑니다. 이는 네트워크 I/O이므로 호출 즉시 **코루틴 A**가 중단됩니다. 

- **코루틴 B**로 넘어가 실행됩니다. **코루틴 B** 역시 토큰 갱신 작업에 들어갑니다. 

- **코루틴 C**로 넘어갑니다. **코루틴 C**도 토큰을 갱신하느라 중단됩니다.

- **코루틴 A**가 재개되고, 새로운 토큰을 DataStore에 업데이트 하는 작업에 들어가서 **코루틴 A**가 중단됩니다.

- **코루틴 B**도 **코루틴 B**에서 생성한 토큰을 DataStore에 업데이트 하는 작업에 들어가, 중단됩니다.

- **코루틴 C** 역시 **코루틴 C**에서 발급한 토큰을 DataStore에 업데이트 하기 위해 중단됩니다.

- 여기서 **코루틴 A**가 새롭게 업데이트한 토큰 정보가 B와 C에서 또다시 변경됩니다!

- **코루틴 A**가 재개되고 DataStore에서 토큰을 가져오는데, 이는 A, B, C가 업데이트 한 토큰 정보 중 하나를 반환합니다(**최신 정보를 보장하지 않습니다**).

- 여기서 발생하는 문제는, **코루틴 A**가 받은 토큰이 Redis에 최종 저장된 토큰 정보와 다를 수 있다는 점입니다.

이를 이용해 HTTP Request를 해봐야 권한 오류가 재차 발생할 것입니다.

즉 앞에서 말한 **토큰 갱신 후 저장 로직은 무조건 동기적(순차적)으로 진행되어야 합니다.** 그래야 `401 Unauthorized` 오류가 발생하였을 때 단 한 번만 토큰을 발급받고, 이 토큰을 나머지 코루틴 실행 시 재사용 할 수 있습니다. `Mutex` 혹은 `runBlocking` 등을 사용하여 해당 블록의 원자성을 보장할 수 있습니다. 그러나 `Mutex`는 suspend 혹은 코루틴 빌더 블록 내부에서만 호출할 수 있기 때문에, 사용자 정의 메서드가 아닌 이미 정해진 메서드(`intercept()`)를 오버라이딩하는 경우에는 사용하기 애매했습니다. 그래서 `runBlocking`으로 현재 실행 중인 스레드(메인)를 blocking하여 해당 블록이 끝날 때까지 나머지 코루틴의 실행을 대기시켰습니다.

```kt
@Singleton
class AuthInterceptor @Inject constructor(
    context: Context,
    private val authDataStore: AuthDataStore,
    @UDIDInterceptor private val udidInterceptor: Interceptor,
    @StethoInterceptor private val stethoInterceptor: Interceptor
) : Interceptor {

    private val gson = GsonBuilder().setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSXXX").create()
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
            throw RuntimeException()
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
        val request = chain.request()
        val token = runBlocking(Dispatchers.IO) {
            authDataStore.getTokenInfo().first()
        }
        val response = chain.proceedWithToken(request, token.accessToken)
        if (response.code != StatusCode.AUTHENTICATE_FAILED.code) {
            limitCnt = 0
            return response
        }
        limitCnt++
        // newToken을 가져올 때까지 메인 스레드를 block
        // 메인 스레드에 속한 코루틴 빌더는 모두 대기 상태가 됩니다.
        val newToken = runBlocking(Dispatchers.IO) {
            val tokenResult = authDataStore.getTokenInfo().first()
            val maybeUpdatedToken = tokenResult.accessToken
            when {
                tokenResult.refreshToken.isEmpty() -> null
                maybeUpdatedToken != token.accessToken -> maybeUpdatedToken
                else -> {
                    val authService = retrofit.create(AuthService::class.java)
                    val refreshTokenResponse =
                        authService.fetchDefaultTokenRes(
                            DefaultToken(
                                key = tokenResult.refreshToken,
                                provider = "REFRESH-TOKEN"
                            )
                        )
                    val code = refreshTokenResponse.code()
                    if (code == StatusCode.SUCCESS.code) {
                        refreshTokenResponse.body().also { tr ->
                            if (tr != null) {
                                authDataStore.saveTokenInfo(tr.copy(provider = tokenResult.provider))
                            }
                        }?.accessToken
                    } else if (code == StatusCode.AUTHENTICATE_FAILED.code) {
                        null
                    } else if (code == StatusCode.FORBIDDEN.code) {
                        authDataStore.popUpBadGateWayDialog(Event(Unit))
                        null
                    } else {
                        null
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

### 정리

다수의 HTTP Request에 의해 코루틴이 생성되면, 이는 비동기로 실행됩니다. 토큰을 갱신하고 DataStore에 업데이트 하는 과정이 비동기로 이루어지면, 중단된 코루틴이 가지고 있는 토큰 정보와 가장 최신 업데이트 된 토큰 정보가 일치하지 않는 문제가 발생할 수 있습니다. 따라서 토큰 갱신 및 업데이트는 **동기적으로 실행되어야 합니다.** `AuthInterceptor` 클래스의 `intercept()` 메서드는 사용자 정의 메서드가 아니기 때문에 `Mutex`를 써서 원자성을 확보하기에 어려움이 있습니다. 왜냐하면 `Mutex.withLock()` 메서드는 suspend 혹은 코루틴 블록 내부에서 실행되어야 하기 때문입니다. 그리하여 코루틴 블록 외부에서 작동하는 `runBlocking`으로 현재 스레드를 block하여 동기적 실행을 보장하였습니다.