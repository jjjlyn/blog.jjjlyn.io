---
title: "Coroutines & Flow"
date: 2022-11-18T13:39:56+09:00
draft: false
categories:
- Android
tags:
- Android
---

비동기(asynchronous) 처리에 주로 사용하는 Coroutines & Flow에 대해 알아보겠습니다. 하단 참고란에 명시한 서적과 작성자의 개발 경험을 토대로 이해한 것에 대해서만 기술합니다.

## 코루틴 왜 사용하는가? 

스레드보다 비용이 저렴합니다!</br>
스레드 중심적인 프로그래밍에서는 많은 스레드를 생성하는 것에 대한 부담이 있습니다. 스레드는 heap 공간을 꽤 많이 소비합니다. 게다가 스레드 간의 컨텍스트 스위칭이 발생하면 이에 따른 오버헤드가 생깁니다.

그러나 코루틴은 스레드에 비해 가성비가 좋습니다. 왜 Why? 

(작성 중)

또한 비동기 코드를 동기 코드처럼 보이도록 하여 Callback Hell에서 벗어날 수 있습니다.

## 코루틴의 역사

### 선점 방식과 협조 방식의 멀티태스킹

프로그래밍에서 멀티태스킹은 프로그램들이 어떻게 한번에 여러가지 작업을 수행하는 것처럼 보이게 하는가를 의미합니다. 주요 관점으로는 협조(Cooperative) 방식과 선점(Preemptive) 방식이 있습니다.

현대 프로그래머들은 멀티태스킹이라고 하면 선점 방식을 먼저 떠올릴 것입니다. 이 방식은 어떠한 멀티태스킹 방법으로 작동하는지 잊어버리기 쉽도록 구현되어 있습니다. 태스크가 실행/실행되지 않도록 준비하는 작업은 **OS가 도맡아 처리**하기 때문입니다. OS가 프로세스와 스레드 스케쥴링을 통해 CPU core에 올리고, 다른 프로세스나 스레드로 컨텍스트 스위칭(context switching)하는 것을 보장하기 때문에 멀티태스킹의 한 방식이라는 것 조차 잊게 되는 것입니다.(개발자 개인이 따로 처리할 일이 별로 없음)

그러나 협조 기반으로 작동하는 멀티태스킹도 있습니다. 어떤 작업이 CPU 시간을 필요로 할 때 이 작업으로 컨텍스트 스위칭을 하는 것입니다. 이는 *코드 상에서 명시적으로 yield(양도)를 하겠다고 명시*해야 합니다. 특정 프레임워크는 yield point에서 태스크를 전환하는 것을 지원하기도 합니다. 16-bit 윈도우 프로그램이나 원조 Mac OS는 이 방법을 사용했습니다.

최근에 협조 멀티태스킹은 프로세스 내부 **프레임워크**에서 주로 사용합니다. OS는 선점 방식을 채택하여, 프로세스의 잘못된 행동을 관리해 줍니다.

**즉 선점 방식은 time scheduling 알고리즘을 이용하여 일정 시간동안 CPU 자원을 사용할 수 있도록 프로세스에 분배하는 것입니다. 짧은 시간동안 빈번하게 배분합니다. 이는 OS가 주관하기 때문에 스레드 A에서 스레드 B에 CPU 자원을 양보(yield)할 것이라는 것을 코드 단에서 명시할 필요가 없습니다.**

**협조 방식은 특정 태스크가 CPU 자원이 필요한 경우(READY 상태) 현재 실행 중인 태스크에서 CPU 자원을 양보하는 것입니다. 이는 코드 상에서 태스크 A에서 태스크 B로 자원을 양도한다는 것을 명시해야 할 필요가 있습니다. 코틀린은 언어상 suspend를 통해 타 태스크로 CPU 자원을 양도할 시간이라는 것을 명시합니다.**

### Green and Red

가끔 프레임워크가 선점형 멀티태스킹을 제공하는 것과 같은 착각(효과)을 주지만 실제로는 협조 멀티태스킹 모델을 채택하는 경우도 있습니다.

초기 자바의 경우, 스레드는 JVM이 관리했습니다. JVM이 스레드 간의 코드 실행을 스위칭하는 방식입니다. 코드에서 각 스레드는 타 스레드에 코드 실행을 양도한다는 것을 명시할 필요가 없습니다. 왜냐하면 JVM이 코드 실행의 책임을 갖고 있어, 자기 자신(JVM)의 '스레드들'을 스위칭하는 것이기 때문입니다. -> 여기서 의문: 그렇다면 JVM은 time scheduling을 사용하지 않는가? *OS의 관점에서 이는 협조 멀티태스킹이라고 볼 수 있습니다. 그러나 자바 프로그래머 관점에서는 선점 멀티태스킹인 것입니다.*

이런 방식을 소위 "Green Threads"라고 하는데, 지금은 Thread가 OS Thread로 매핑되는 "Red Threads"(-> Native Thread라는 건가? 1:1 유저-커널 스레드 매핑을 말하는건가?)로 대체되었습니다. Browser Applets(플러그인 같은 작은 응용 프로그램)이나 스윙 데스크탑 앱(이게 뭐지?) 등에서 웹 앱으로 자바 사용처가 변경되면서 CPU 파워의 사용을 최대한 극대화(멀티코어 프로세서를 최대한 사용)하는 병렬 프로세스가 중요해졌습니다.

> 여기서 Green Threads가 무엇일까요?
> 
> 커널 위 유저 공간에 JVM이 올라가면 JVM이 여러 개의 스레드를 생성합니다. 이를 Green Threads라 합니다. Green Threads는 **커널 스레드 한 개**와 연결됩니다. 하나의 스레드만 커널에 접근할 수 있기 때문에 (N:1 모델이기 때문에 여러 개 스레드 중 단 한 개의 유저 영역 스레드만 일정 시간동안 커널 스레드와 매핑됩니다.) 인터럽트가 발생하면 blocking 돼서 나머지 스레드가 일을 할 수 없게 됩니다. 멀티코어 아키텍처에서는 비효율적입니다.

### 코루틴의 컨셉

코루틴은 협조 방식 컨셉입니다. 코루틴은 다른 코루틴에 실행을 양도할 수 있는 특징을 가지고 있어, 언제든 바로 실행 가능한 상태(Ready)의 코루틴이 대기하고 있으면 이를 즉시 실행할 수 있습니다. 또한 코루틴은 다른 코루틴이 실행되는 것을 기다릴 수 있습니다.

전통적인(?) 코루틴에서는 yield 키워드를 그대로 사용합니다. 코루틴에서는 suspend 키워드를 사용하고, 이를 만나는 시점에서 양도(다른 코루틴에 코드 실행을 넘기는 행위)를 자동적으로 수행합니다. 즉 프로그래밍 관점에서 개발자는 yield control에 대해 직접 생각할 필요가 없습니다. 

코루틴은 여느 협조 방식을 채택하는 멀티태스킹과 같이, 협조를 필요로 합니다. 예를 들어 코루틴이 다른 코루틴에 실행을 양도하지 않을 시 다른 코루틴들은 일정 시간동안 실행할 수 없게 됩니다. 그래서 코루틴의 협조 방식은 앱(프로세스) 간의 관계에서는 좋은 선택이 아닐 수 있습니다. 왜냐하면 앱 개발자들은 본인의 앱이 타 앱에 비해 중요도가 높다고 생각하기 때문입니다. 그러나 앱 내부에서는 코루틴 간의 협조가 중요합니다. 코루틴 하나가 잘못 동작하는데 다른 코루틴들이 이를 돕지 않으면 앱 크래시 혹은 버그가 발생할 수 있기 때문입니다. 즉 코틀린 코루틴은 in-app에서 유용한 동시성 모델이라고 볼 수 있습니다. 여전히 코틀린은 OS와 선점 멀티태스킹 방식에 의존하긴 합니다.(앱 간의 관계 차원에서 - 프로세스, 스레드)

### 다시 주목받기 시작한 코루틴!!

멀티스레드 프로그래밍이 각광받던 시기에 코루틴은 잠시 자취를 감추었습니다. 그러나 멀티스레드 방식의 복잡성이 밝혀지고 리액티브 프로그래밍이 대두되면서 다시 인기를 얻습니다.

코루틴은 작업(코루틴)과 그 작업을 실행할 환경(스레드 혹은 스레드 풀)을 분리합니다. 코루틴이 작업할 스레드 혹은 스레드 풀을 지정할 수 있습니다. 즉 안드로이드 메인 스레드가 UI 업데이트에만 사용되게 제한하는 등의 문제를 해결할 수 있게 되었습니다. 

코루틴이 Native Thread에서 실행되든 JVM의 Green Thread에서 실행되든, 이는 코루틴 작성자(개발자)가 아니라 코루틴 라이브러리에서 결정합니다. 코루틴은 스레드와 스레드 풀을 공유하기 때문에 개발자는 실제 사용할 수 있는 스레드보다 훨씬 많은 코루틴을 사용할 수 있습니다.

## CoroutineScope

모든 코루틴 작업은 CoroutineScope가 관리합니다. 이를테면 코루틴이 더 이상 필요하지 않을 때 이를 취소하거나 정리하는 작업을 합니다. CoroutineScope는 개발자가 생성한 코루틴을 추적하여, 실행된 코루틴을 모두 취소할 수 있습니다.

GlobalScope는 프로세스의 생명주기를 따라가기에 가장 수명(생명주기)이 깁니다. 공식 문서 등에서 예시를 들 경우에는 GlobalScope를 사용하는 것이 적합하지만(어디서든 쉽게 실행 및 접근 가능), 실제 개발 시에는 더 범위가 작은 Scope가 필요합니다. 가령 UI를 업데이트하기 위해 I/O를 할 때 다른 화면으로 전환하면 더 이상 이 코루틴 작업이 필요없게 됩니다. 안드로이드에서 Jetpack은 이를 위해 다양한 범위에 걸친 CoroutineScope 구현체를 제공합니다.

코루틴 스코프는 범위에 속한 다수 코루틴의 *구조화된 동시성*을 지원합니다. 예를 들어 코루틴 스코프 내부의 특정 코루틴이 crash될 때 나머지 코루틴은 모두 취소됩니다. 

> 코루틴 스코프는 어떻게 생성될까요?
> 
> - GlobalScope
> - 코루틴 빌더를 실행할 때
> - 프레임워크가 지원하는 스코프를 사용할 때 (e.g. 안드로이드 `viewModelScope()` - ViewModel의 생명주기를 따라갑니다.)
> - `withContext()`를 호출할 때 - 현재 dispatcher를 변경하여 다른 스레드 풀에서 코루틴을 실행하고자 할 때
> - `coroutineScope()`를 호출할 때 - 현재 dispatcher를 유지한 채 새로운 코루틴 스코프를 생성하고자 하는 경우
> - `supervisorScope()`를 호출할 때 - 코루틴 스코프 내부에서 돌아가는 코루틴 하나가 exception이 나도 나머지 코루틴을 취소하지 않고 실행하고자 할 때
> - 테스트 스코프(Test Scopes)
> - 커스텀 스코프(Own Custom Scopes)

<!-- ### viewModelScope

viewModelScope에서는 코루틴 스코프가 ViewModel의 생명주기를 따라갑니다. ViewModel의 onCleared() 호출 시 코루틴 스코프도 같이 취소(cancel)됩니다. 

ViewModel을 사용하면 View(Activity, Fragment)에 Configuration Changes가 발생하여, View가 다시 `onCreate()` ~ `onDestroy()`의 과정을 거쳐도 데이터를 그대로 유지할 수 있습니다. ViewModel은 기본적으로 생명주기가 깁니다. Activity 혹은 Fragment의 `onDestroy()`가 호출되어야 비로소 ViewModel도 제거됩니다.  -->

<!-- Single Activity Application + AAC Navigation을 사용하는 경우 보통 Activity(MainActivity)위에 NavHostFragment가 올라가고, 최종적으로 실제 사용자가 보는 View(Fragment)가 올라간다. 마지막에 올라가는 Fragment가 NavHostFragment 위에서 계속 교체되는 형태라고 할 수 있다. 그래서 앱을 종료하기 전까지 Activity의 `onDestroy()`가 불릴 일은 없다고 보면 된다. 그러면 Fragment의 생명주기를 따라가고 싶은 ViewModel은 Fragment의 생명주기를 신경써야 한다는건데 보통 Fragment A(현재 화면)에서 Configuration Changes(e.g. 폰트 변화, 화면 전환 등)이 발생할 경우 A Fragment는 `onDestroy()`가 불리고 새롭게 `onCreate()`를 호출한다. A Fragment의 생명주기를 따르는 ViewModel의 경우 A의 `onDestroy()`가 호출된 이후 `onCleared()`가 불리는데 그러면 어떻게 상태 관리가 되는 것인가? -> ComponentActivity의 inner static class인 NonConfigurationInstance가 관리하는데 여기 주제에서 벗어남 -->

## Coroutine Builder

코루틴 빌더를 통해 코루틴이 실행됩니다. 코루틴 빌더는 람다 표현식을 사용합니다. 코루틴이 수행할 실제 작업을 람다 내부에 작성하면 됩니다.

종류로는 `launch()`, `async()`, `runBlocking()` 등이 있습니다. (Kotlin/JVM 라이브러리 기준)

**launch()**

*fire and forget* - 한번 실행하고 나서 잊어버린다.

한번 실행하고 나면 람다의 결과를 반환하지는 않습니다. `launch()`는 Job 객체를 반환합니다. 이는 현재 실행 중인 작업을 관리(e.g. cancel)하는데 사용합니다.

**async()**

Job 객체의 subclass인 Deffered가 반환됩니다. `launch()`가 람다의 결과를 무시(ignore)하는 것과 달리, async는 **Deffered 객체 안에 람다의 결과를 담아 반환**합니다. 직접적인 반환값을 필요로 할 때 사용합니다. `await()`으로 async 람다의 결과를 기다린 후 Deffered 객체를 받습니다. 

참고로 `await()` 자체도 suspend 함수이기 때문에 코루틴 빌더 혹은 다른 suspend 함수 내에서 호출해야 합니다.

**runBlocking()**

suspend 함수를 코루틴 밖에서 호출하고 싶을 경우 사용합니다. 나만의 코루틴을 설정할 수 없는 경우(그런 권한이 없는 곳에서 e.g. 안드로이드 프레임워크 메서드)에 `runBlocking()`을 호출하면 코루틴 launcher를 block합니다.(모든 코루틴의 실행을 block 상태로 만드는 것) `runBlocking()`도 람다를 제공하여 코루틴을 그 내부에서 실행할 수 있습니다. 그러나 코루틴 작업이 완료되고 나서야 비로소 반환됩니다.(block이 풀림) **즉 suspend를 만났다고 다른 코루틴으로 갈아타는 것이 아니라 계속 대기 상태로 있게 됩니다.**

## suspend Function

suspend 키워드는 코틀린 언어 차원에서 지원합니다. 그 밖의 것들은 라이브러리의 도움을 필요로 합니다.

자바 개발자 입장에서는 Runnable 위에 Executor(`post()`)가 올라가 실행되는 것과 유사하다고 생각하면 됩니다. Dispatchers.Main이라면 안드로이드 개발자 입장에서 View 안에서 Runnable을 `post()`로 감싸서 Runnable이 메인 스레드에서 실행되게 하는 것과 비슷합니다.

그렇다고 완전히 동일한 것은 아닙니다.

Runnable 시나리오에서는 지정된 스레드(혹은 스레드 풀)의 작업 단위가 Runnable 그 자체입니다. 스레드가 Runnable의 코드를 실행하면, 이 스레드가 작업 중으로 바뀌기 때문에 또 다른 Runnable 객체를 넘겨줘도 첫번째 Runnable 작업이 종료될 때까지 기다려야 합니다.

그러나 코틀린은 suspend 키워드를 제공합니다. 코루틴 시스템(현재 스레드)에서는 *현재 실행 중인 코드 블럭을 중지하고* 동일한 Dispatcher를 갖고 있는 다른 코루틴 빌더의 코드로 넘어갈 수 있습니다.(넘어가려는 코루틴 빌더에 실행 가능한 코드가 있다는 전제하에)

## CoroutineContext

이름 그대로 코루틴을 실행하는 컨텍스트를 말합니다. 코루틴 빌더에 제공되는 **Dispatcher**는 CoroutineContext의 요소 중 하나이며, **Job**이라는 것도 있습니다.

`withContext()`(최상위 함수 - 클래스 밖에 위치한 전역함수)는 suspend 함수입니다. 즉 다른 suspend 함수 내부나 코루틴 빌더 내부에서만 실행될 수 있습니다. `withContext()`는 기존 코드가 사용하는 Dispatcher에서 다른 Dispatcher로 갈아타 변경된 CoroutineContext에서 실행됩니다. 만약 `withContext()`를 호출한 현재 블록의 Dispatcher가 Dispatchers.Main 이라면, `withContext()`가 다른 스레드(풀)에서 실행되는 동안 Dispatchers.Main의 현재 코드 블록 실행은 중지될 것입니다(`withContext()`가 suspend 함수이기 때문). `withContext()`가 결과값을 반환하기까지 코틀린은 Dispatchers.Main에서 READY 상태에 있는 다른 코루틴들 중 하나를 선택해서 실행할 수 있습니다.

### Dispatcher

코루틴 빌더에 해당 코루틴이 특정 스레드(풀)에서 실행될 수 있도록 설정을 주는 역할을 합니다.
CoroutineContext는 Dispatcher, Job 등의 요소(element)를 포함합니다.

- Dispatcher는 **코루틴을 실행하는 환경**입니다.
- Job은 **코루틴 그 자체**를 의미합니다.

**일반적으로 제공되는 Dispatchers**
- Dispatchers.Default은 **block될 일이 없는** 일반적인 백그라운드 작업을 할 때 유용합니다.
- Dispatchers.IO는 **block될 가능성이 있는** I/O 관련 백그라운드 작업을 할 때 사용합니다.
- Dispatchers.Main은 메인 스레드가 필요한 작업을 할 때 사용합니다.

    **Dispatchers.Default**

    코루틴 빌더에 별다른 Dispatchers 설정을 주지 않을 시 기본값이 됩니다. 일반적인 백그라운드 작업을 수행할 때 사용합니다.

    **Dispatchers.IO**

    잠재적으로 block될 가능성이 있는 I/O 작업에 사용합니다. Dispatchers.Default와 스레드 풀을 공유합니다. 스레드 풀을 사용하는 로직은 Dispatchers.Default와 차이가 있습니다.

    즉 Dispatchers.Default와 Dispatchers.IO 모두 백그라운드 작업에 사용하나, 전자는 block될 가능성이 없는 작업, 후자는 block될 여지가 있는 작업을 대상으로 합니다.

    **Dispatchers.Main**

    *magic thread*로 불리는 메인 스레드에서 코루틴이 돌아갑니다. UI 작업을 할 때 사용합니다.

**일반적이지 않은 Dispatchers**

- `newFixedThreadPoolContext()`

- `newSingleThreadContext()`

**Dispatcher 결정**

Dispatcher를 결정하는 것은 스레드(풀)을 선택하는 것과 비슷합니다.

안드로이드 개발자 입장에서 `launch(Dispatchers.Main)`을 Handler 혹은 View에서 호출할 수 있는 `post()`와 비슷하다고 보면 됩니다. 이 메서드는 Runnable 객체를 갖고 있으며 호출할 때마다 메인 스레드의 작업 queue에 하나씩 추가됩니다. 즉 `post()`를 즉시 호출해도 Runnable을 실행할 순서에 아직 도달하지 못했기 때문에 즉시 실행되지 않습니다. 코루틴도 이와 마찬가지로 `launch(Dispatchers.Main)`을 호출한다고 그 즉시 실행되는 것이 아닙니다.

Dispatchers.Main.immediate라는 특수한 Dispatcher가 있긴 한데, 이걸 사용하면 호출하는 그 즉시 실행됩니다.

안드로이드 개발자 입장에서 `launch(Dispatchers.Main.immediate)`는 `runOnUiThread()`를 호출하는 것과 같다고 생각하면 됩니다. `post()`와 같은 역할을 하는 것처럼 보이지만 약간의 차이가 있습니다.

- `post()`는 항상 Runnable을 작업 queue 넣습니다.
- `runOnUiThread()`는 현재 실행하고 있는 스레드가 메인 스레드인 경우 Runnable을 그 즉시 실행합니다. 만약 다른 스레드에서 작업하고 있는 경우 `runOnUiThread()`는 `post()`와 동일한 행동을 취합니다.(work queue에 Runnable 객체를 추가하는 행위)

### Jobs

코루틴을 시작하는 것은 코루틴 빌더의 람다 블록 내부의 코드를 실행해달라고 시스템에 요청하는 것과 같습니다. 이 코드는 바로 실행되지 않고 queue에 들어갑니다.

Job은 queue에 들어간 코루틴을 조작하는 손잡이(handler)입니다. 서로 다른 Job의 실행 순서를 관리(join, parent-child relation)하거나, 상태(new, active, completed, cancelled)를 감지하거나 이에 변화를 줄 수 있습니다.

## Flows and Channels

### Hot and Cold Stream

- GPS 위성은 지구 상에 GPS 수신기가 켜진 상태가 아니더라도 계속 송신합니다. 이렇게 *수신 여부와 상관없이 계속 데이터를 송신하는 것*을 Hot Stream이라고 합니다.
- 스마트폰의 GPS 라디오 기능은 배터리를 절약하기 위해 기본적으로 turn-off 상태입니다. 앱이 GPS 상태의 업데이트를 요청할 경우에만 GPS 정보를 송신합니다. *수신 대상이 정보를 요청할 때만 데이터를 송신하는 것*을 Cold Stream이라고 합니다.

기본적으로 Flow는 Cold Stream, Channel은 Hot Stream 기반입니다. **Hot Stream**을 위한 Flow(**SharedFlow, StateFlow**)도 존재합니다.

### Flows

- 코틀린 코루틴 시스템에서 제공하는 최상위 함수를 호출합니다. 
  ```kt
  flow{

  }
  ```
- Channel을 Flow로 변환합니다.
- 서드파티 라이브러리를 통해 Flow를 가져옵니다.

**flow 블록으로 생성한 Flow는 단 한번만 collect할 수 있습니다.** (Each Flow created by flow() can only be collected once — we cannot have multiple collectors.)

**반면 StateFlow, SharedFlow는 구독자 모두가 동시에 데이터를 받을 수 있습니다.**

MutableSharedFlow는 캐싱의 기능도 있습니다. `reply(N)`은 가장 최근에 내보낸 N개의 데이터를 캐싱한다는 의미입니다.

### Channels

- `produce` 블록을 사용(람다 표현식)
  
  **ReceiveChannel**을 반환합니다.
- `Channel()`을 사용할 수 있습니다. `Channel()`은 생성자처럼 보이지만 Channel이 인터페이스라, 실제로는 Channel 객체를 만드는 최상위 factory 함수를 호출하는 것입니다.

  **Channel.BUFFERED** 파라미터 옵션은 64 elements(기본값) 버퍼를 의미합니다. 만약에 `send()`를 호출하고 버퍼에 공간이 있으면, `send()`는 버퍼에 전달된 데이터를 추가한 후 즉시 값을 반환합니다. `send()` 호출 후 버퍼가 다 찬 상태라면, `send()`는 구독자(consumer)가 Channel 버퍼에 쌓인 데이터를 모두 소비(receive)할 때까지 중단됩니다.
- `actor` 블록을 사용(`produce`와 반대)
  
  **SendChannel**을 반환합니다. 

## SharedFlow와 StateFlow

### SharedFlow

- 브로드캐스트 매커니즘 

  Flow로 내보낸 데이터를 동시에 모든 구독자에 공유할 수 있습니다.

- 캐시 매커니즘

  가장 마지막으로 내보낸 N개의 데이터를 보관합니다. 단 추가적으로 설정해야 합니다.

일반 Flow와 달리 SharedFlow는 hot stream입니다. 이는 Flow를 생성하자마자 그 즉시 실행되는 것을 의미합니다. 얼마나 많은 구독자가 있든 간에 내보낼 이벤트가 있으면 이를 사용하지 못하게 되더라도(버려지더라도) 무조건 내보냅니다. 그래서 SharedFlow를 사용할 때는 생성과 이벤트 구독 순서를 유념해야 합니다.

MutableSharedFlow를 만들 때 `MutableSharedFlow()`를 직접 호출하는 것 외에도 또 다른 방법이 있습니다. 일반 Flow로 시작하고 `sharedIn()` 메소드를 호출하면 됩니다.

정해진 크기의 이벤트가 있고 끝나는 시점에 신호를 보내주는 기능보다는, 구독자가 더 이상 필요로 하지 않을 때까지 정보를 연속적으로 공급하는 기능이 필요할 때 SharedFlow가 강력한 방법이 되겠습니다.

SharedFlow는 영원히 종료되지 않기 때문에 특정 연산기능(operators)은 아무 효력을 발휘하지 못합니다. Flow의 컨텍스트를 변경하거나 cancellable, buffered를 사용한 새로운 Flow 생성 등은 SharedFlow에 사용해봐야 아무 효과가 나타나지 않습니다.

SharedFlow는 Broadcast type of communication을 제공하기 때문에 **실패하거나 완료될 수 없습니다.** 사용자가 직접 Flow 혹은 CoroutineScope를 취소하기 전까지는 계속 정보를 전송합니다. 그렇기 때문에 Flow의 이벤트 전송을 더 이상 받지 않으려면 Flow를 정리(clean up)하는 것이 좋습니다.

### StateFlow

SharedFlow의 모든 기능과 가장 최근에 내보낸 데이터에 대한 저장(캐싱) 기능을 포함합니다.

- `.value`로 가장 최근 보낸 데이터에 접근할 수 있습니다. 직접 데이터에 접근하면 특정 시점의 StateFlow 데이터를 가져옵니다. 그러나 이 방법보다는 자주 변하는 데이터면 StateFlow를 구독하는 편이 낫습니다.
- `collect`를 이용하여 구독할 수 있습니다(변화 감지).

- **특정 한 곳에서만 업데이트 하는 상황이 보장될 때** `setValue()`(`.value`)로 업데이트 할 수 있습니다.
- `tryEmit`를 사용하면 blocking 혹은 suspending할 필요가 없습니다. 데이터를 업데이트 하기 가장 안전한 방법입니다. 그러나 여러 차례에 걸쳐 데이터의 변화가 나타날 때 가장 최신의 데이터로 업데이트 된다는 보장이 없습니다.
- `emit`을 사용하면 코루틴 빌더 안에서 실행하거나 suspend 함수 내부에서 돌려야 합니다. 버퍼가 다 찼을 시 중단될 수 있습니다. 최신의 데이터로 업데이트하기에 가장 확실한 방법입니다.
  
SharedFlow와는 Backpressure(데이터를 내보내는 속도가 너무 빠르거나, 받는 속도가 너무 느려서 중간에 데이터가 손실되는 상황)를 처리하는 방법에서 차이가 있습니다. SharedFlow가 여러 종류의 옵션을 제공하는 것에 반해, StateFlow는 단 하나의 옵션만을 제공합니다. 과거의 객체를 새로운 객체로 교체하는 방식으로, 이를 **Conflation**이라고 합니다.

StateFlow는 **single-element 버퍼**를 갖고 있습니다. 그래서 바로 최근에 업데이트된 객체를 계속 갖고 있는 것인데, 다시 업데이트하면 기존의 것이 새로운 객체로 교체됩니다. 
상태 관리에 좋습니다. 그래서 보통 sealed class를 StateFlow 객체로 많이 사용합니다.

StateFlow는 내용(content)의 일치성(equality)으로 conflation 여부를 판단합니다.

**Q. LiveData와 차이는?**

- Dispatcher Control

  **LiveData는 메인 스레드에서만 관찰할 수 있습니다.** LiveData가 다른 특정 스레드에서 데이터를 내보내도록 하는 기능은 애초에 제공하지 않습니다. UI 계층에서 데이터를 받는 경우에는 상관 없으나, Room DB의 DAO가 LiveData를 반환하는 경우 UI 계층에 도달하기 전까지 이를 구독해서는 안됩니다. DAO와 UI 사이에는 Transformation과 MediatorLiveData가 필요한데, 이는 LiveData가 내보내는 결과값을 UI 계층에서 필요한 데이터 양식으로 조작하기 위해 사용합니다. 동시에 메인 스레드 외에 다른 장소에서 stream이 소비되지 않도록 해야 합니다.

  StateFlow는 Dispatchers를 이용해 메인 스레드 외 장소에서도 데이터를 내보내거나 관찰할 수 있습니다.

- 더 많은 연산기능(operators) 제공
  
  LiveData가 Transformations에서 `map()`과 `switchMap()`만 제공하는 반면, StateFlow는 기본적으로 Flow이기 때문에 다양한 연산기능을 제공합니다. 

- Scope Flexibility

  LiveData는 안드로이드 라이프사이클을 잘 알고 있습니다.

  (작성 중)

**Q. Single Event - Channel vs SharedFlow or StateFlow??**

SharedFlow은 Channel과 마찬가지로 hot stream 기반이며 ConflatedBroadcastChannel과 같은 방식으로 동작하지만, 보다 더 간편한 API를 제공합니다. 또한 다수의 구독자가 동일한 stream을 공유할 수 있습니다. 반면 Channel은 다수의 구독자에 데이터를 내보낼 수 없습니다. 하나의 이벤트를 다수의 구독자에 내보내고 싶으면 **SharedFlow**를 선택하면 될 것이고, **단독 구독자**인 것이 보장된 경우 **Channel**을 쓰면 됩니다. 한편 **StateFlow를 쓰는 것은 권장하지 않습니다.** StateFlow는 다수의 구독자에 상태(State)를 내보내는 것이기 때문에, 내용이 일치(equal)하면 `emit()`이 invoke되지 않을 것입니다. 게다가 이전 상태를 항상 저장하고 있기 때문에(single-element 버퍼) Configuration Changes(e.g. 화면 전환 Portrait -> Landscape 등)발생 시 다시 바로 이전에 내보낸 데이터를 invoke(re-deliver)합니다. 따라서 일회성 이벤트를 내보내는 데 사용하기에는 부적절합니다.

<!-- ## Bridging to Callback APIs

`suspendCoroutine()`과 `suspendCancellableCoroutine()`은 one-shot 비동기 작업에만 사용할 수 있습니다. -->

## 참고

**전문 서적**

- [Elements of Kotlin Coroutines, Version 0.3](https://www.goodreads.com/book/show/51171222-elements-of-kotlin-coroutines)
- [Kotlin Coroutines by Tutorials (Third Edition): Best Practices to Create Safe & Performant Asynchronous Code With Coroutines](https://www.amazon.com/Kotlin-Coroutines-Tutorials-Third-Asynchronous/dp/1950325687)