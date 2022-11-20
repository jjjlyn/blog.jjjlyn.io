---
title: "Android Architecture"
date: 2022-11-18T13:39:56+09:00
draft: false
tags:
- Android
---

## 들어가기에 앞서
**본 문서에서 출처를 명시하지 않은 첨부 이미지는 작성자가 직접 제작한 작업물입니다.</br>
2차 가공, 공유 등을 하여도 문제되지 않으나 내용에 오류가 있을 수 있으므로 권장하지 않습니다.</br>
잘못된 내용으로 발생하는 피해에 대해서는 책임을 지지 않습니다.**

> 안드로이드 앱은 익숙해지면 찰리 채플린이 컨베이어 벨트에서 나사를 조이는 것 마냥 반복적으로 찍어낼 수 있게 됩니다. 그러나 전체 플랫폼의 동작 원리를 이해하는 것은 여전히 쉽지 않습니다. 시스템 관련한 비슷한 주제의 책들을 몇번이나 뒤적여도 뒤돌면 잊어버리더군요. 아무래도 확실한 이해가 부족한 탓인 것 같습니다. 공부한 내용을 숙지하고, 문서를 지속적으로 개선하자는 차원에서 기록합니다. 제가 이해하는 선에서 최대한 정리하였으나 간혹 틀린 내용이 있을 수 있습니다.

(안드로이드 환경에 맞게 커스터마이징한) 리눅스 커널부터 안드로이드 어플리케이션 계층까지를 안드로이드 플랫폼이라고 볼 수 있는데, 편의상 통틀어서 안드로이드 OS라고 해도 틀린 말은 아닙니다. 커널부터 최상위 계층까지 OS가 관장하기 때문입니다.

안드로이드는 모든 시스템 기능을 **서버 프로세스** 형태로 제공합니다. 사용자 어플리케이션과 시스템 서비스 간의 통신을 설명한 그림들을 보면 대부분 계층이 나뉘어진 것처럼 표현한 경우가 많습니다. 안드로이드 입문자 입장에서는 계층이 분리되어 있다는 오해를 살 수도 있는 표현 방식이라고 생각합니다. 그러나 사실 앱 개발자가 개발한 응용 프로그램이나 시스템 서비스나 모두 같은 층의 유저 공간에서 실행되는 리눅스 프로세스에 불과합니다. 보다 더 쉬운 이해를 위해 이미지를 첨부합니다.
![User Space Linux Processes](/images/android/user-space-processes.png)

## Application Framework
어플리케이션 프레임워크는 시스템 서비스, 안드로이드 런타임 등의 집합체 입니다. 

> 보통 처음에 프레임워크가 무엇인가 의문을 가집니다. 범위에서 약간 벗어난 내용이긴 하나, 프레임워크는 라이브러리와 비교하면 쉽게 이해할 수 있습니다. 어떤 코드를 A라고 한다면 내가 짠 코드를 A가 사용하느냐, 아니면 내가 짠 코드에서 A가 돌아가느냐의 차이입니다. 전자에서 A는 프레임워크고 후자의 A는 라이브러리라고 보면 됩니다. 프레임워크(이미 짜인 구조)에 개발자가 내용을 채워 넣으면 이 코드가 기존 시스템에 맞게 사용되는 것입니다.

![Scope Of Framework](/images/android/android-framework.png)

![Overall Architecture System Services](/images/android/android-overall-architecture.png)

### Init
안드로이드 OS의 부팅 과정도 보통의 리눅스와 동일합니다. 커널 부팅 작업의 마지막 단계로 init.rc 설정 파일에 의거하여 init 프로세스를 실행하는 것입니다.(물론 안드로이드 환경에서 init.rc는 그 플랫폼 특성에 맞게 몇 가지 특별한 설정이 추가되어 있습니다.) 여기서 init이란 유저 모드(user space)에서 가장 처음 실행되는 프로세스로, 그 후 생성되는 유저 모드의 모든 어플리케이션의 부모가 됩니다. 

### Zygote
init 프로세스는 `app_process` 명령어를 통해 Zygote의 실행을 위임합니다. `app_process`는 `AndroidRuntime` 객체를 생성하는데, Dalvik VM을 사용하는 환경에서는 이 `AndroidRuntime`이 Dalvik VM의 생성과 실행을 담당합니다. Zygote는 Dalvik VM에 올라가 실행됩니다. ART를 이용하는 환경에서는 VM이 따로 필요없으며, Zygote(자바파일)을 바이너리 파일로 한번에 변환하여 곧바로 실행합니다. 이에 관한 설명은 하단에 보다 자세히 기술하였습니다.

**역할**

Zygote는 자기 자신을 초기화 하면서 안드로이드 프레임워크(*$AOSP/framework*) 전체를 사전에 로드합니다.(런타임에서 필요할 때 동적으로 라이브러리를 로드하는 방식이 아닙니다) 초기화가 완전히 끝난 후 루프(loop)에 들어가 소켓을 열고 안드로이드 시스템으로부터 '새로운 프로세스를 시작한다는 요청'을 기다립니다. 시스템은 새로운 프로세스를 시작할 때 Zygote 소켓에 연결한 후, 어떠한 프로세스가 생성되어야 하는지에 대한 정보가 담긴 작은 패킷을 전송합니다. 

![Zygote creates new process](/images/android/zygote-create-new-process.png)</br>

Zygote는 전송받은 패킷을 바탕으로 자기 자신을 fork(복제)하여 새로운 프로세스를 생성합니다. 

즉 Zygote는 *(1) 안드로이드 환경에서 공통적으로 사용하는 리소스와 라이브러리를 사전에 로드* 하고, *(2) 새로운 프로세스를 생성할 때 자기 자신을 복제* 하는 역할을 합니다. 이 두 가지 기능은 프로세스 실행 시간을 단축하고 OS가 효율적으로 메모리 관리를 할 수 있도록 합니다.

보통 자바 코드로 작성된 데스크탑 어플리케이션을 실행하려면 다음 과정을 거쳐야 합니다.

- OS 자원 할당
- JVM 실행
- Java 라이브러리 lazy load
- 기본 라이브러리 초기화
- 어플리케이션 로드
- 어플리케이션 로드하면서 새롭게 추가된 클래스 로드
- minor GC
- 위 단계가 완료된 후 VM이 어플리케이션의 `main()`을 실행

위에서 언급한 일련의 과정은 배터리로 전원을 공급받는 모바일 기기에서는 치명적일 수 있습니다. 앱 실행 몇 번하다 배터리가 방전되는 사태가 벌어질 수 있습니다. 새로운 프로세스가 실행될 때마다 이 과정을 반복하는 것을 방지하기 위해 나온 해결책이 바로 Zygote입니다.

**동작**

자바 데스크탑 어플리케이션의 shared memory runtime linking을 사용하는 대신, Zygote는 시스템이 시작될 때 안드로이드 프레임워크 전체를 로드합니다. 새로운 어플리케이션을 생성하라는 요청이 들어올 때마다 일단 자기자신을 복제합니다. 

> 운영체제는 일반적으로 외부 단편화를 방지하기 위해 Paging 기법을 이용합니다. 외부 단편화란 쉽게 설명하면, 프로세스의 처음부터 끝까지의 주소를 연속적으로 배치할 때(Contiguous Allocation) 나타나는 메모리 낭비라고 생각하면 됩니다. 이 메모리 낭비를 방지하기 위해 가상 메모리를 동일한 크기(Page)로 잘라 프로세스 자원 주소를 CPU가 임의로 각 Page(자른 덩어리 중 한 개)에 할당한 후 실제 물리 메모리와 매핑해주는 기법을 Paging이라고 합니다. 본 내용은 해당 문서가 다루는 범위를 벗어나기에 이 정도로만 기술합니다.

Zygote 역시 Paging 기법을 사용하는데, 그림으로 나타내면 아래와 같습니다.

![Zygote Structure](/images/android/zygote-structure.png)

Zygote는 **COW(Copy On Write)** 방식을 이용합니다. 새로운 어플리케이션 실행 요청이 들어올 때 Zygote는 자기 자신을 복제한다고 하였습니다. 복사된 자식 프로세스(새로운 어플리케이션)는 페이지 테이블을 새로 할당받는데, 이는 Zygote의 페이지 테이블의 복사본이기 때문에 Zygote가 가리키는 물리적 메모리 주소와 동일한 주소를 참조합니다. Zygote는 실행될 때 안드로이드 프레임워크 전체를 미리 로드하기 때문에 복사본인 자식 프로세스는 이 메모리를 공유해서 사용하면 됩니다. 이는 메모리 절약 측면에서 굉장한 장점이 있습니다.

![Zygote Copy On Write - Read](/images/android/zygote-cow-read.png)

그러나 부모-자식 간에 동일한 물리적 주소를 공유하면 새로운 프로세스만의 고유 영역이 보장되지 않는 문제가 생깁니다. 이런 현상은 '쓰기(write)' 행위로 인해 발생합니다. 이를 방지하기 위해 Copy On Write 방식을 사용합니다. 가령 새로운 프로세스가 공유 자원에 쓰기 작업을 요청한다면, 하드웨어는 커널에 이를 알립니다.(물리 메모리로의 접근을 시도하였으므로) 커널은 가상 메모리 공간에 쓰기 행위의 대상이 되는 페이지의 사본을 만든 후(페이지를 새로 생성), 새로운 프로세스가 이 복사본을 참조하도록 합니다. 이를 통해 두 프로세스는 공용으로 사용할 수 있는 자원은 공유하면서(READ), 쓰기를 수행할 경우는 사본을 만들어 그 곳에 쓰기를 함으로써 기존 자원을 훼손하지 않을 수 있습니다.

![Zygote Copy On Write - Write](/images/android/zygote-cow-write.png)

**Dalvik Runtime에서 Zygote (< Android 5.0)**

![Dalvik Runtime](/images/android/dalvik-runtime.png)

Zygote는 자바 코드로 작성되어 있으므로 궁극적으로 이를 실행하려면 바이너리 코드로 변경해야 합니다. 일단 javac로 바이트코드(.class)로 바꿔주는 작업을 합니다. class파일을 jar로 묶어주고, *dx*툴을 이용하여 DEX 파일로 변환합니다. 변환된 Zygote DEX 파일은 Dalvik VM에서 한 줄 씩 해석하여 실행할 것입니다. 이를 JIT(Just In Time) 컴파일이라 합니다. 사실 따지고 보면 Zygote도 보통의 자바 어플리케이션과 다를 바가 없는 것입니다. 차후 시스템으로부터 새로운 어플리케이션의 생성 요청이 들어올 때 실행된 Zygote 프로세스를 fork한 후 새로운 어플리케이션을 로드하여, 이 역시 Dalvik VM에서 돌아갈 수 있도록 합니다.

**ART Runtime에서 Zygote (>= Android 5.0)**

![ART Runtime](/images/android/art-runtime.png)

ART도 Dalvik VM과 마찬가지로 DEX를 매개언어(Intermediate Language)로 사용합니다. ART가 도입되기 이전에 자바로 작성한 프로그램들(DEX 형식)을 Dalvik Runtime이 한 줄 씩 해석한 후 실행했다면(JIT 컴파일러), ART Runtime은 한꺼번에 네이티브 코드로 변환한 후 실행합니다. 이를 AOT(Ahead of Time) 컴파일이라 합니다. 비록 JIT에서 AOT로 컴파일 실행 방식이 바뀌었지만, **DEX 파일을 바이너리 코드로 변환·실행하는 근간**은 유지되는 것이기 때문에, 기존 프로그램들은 이에 영향받지 않고 정상적으로 동작합니다. 이렇게 특정 환경이 크게 바뀌어도 기존 시스템이 정상적으로 작동할 수 있는 상태를 '이식성(portability)이 높다'고 표현합니다. DEX와 같은 매개언어는 이 이식성을 높이기 위해 사용하는 것입니다.

ART에서 DEX를 바이너리 파일로 변환하는 도구를 *dex2oat*라고 합니다. *dex2oat*는 여러 하드웨어에서 돌아갈 수 있도록 각종 버전을 제공하고 있고, 이는 안드로이드 기기에 OS의 한 부분으로서 설치됩니다. *dex2oat*가 각 하드웨어에 맞추어 바이너리 코드로 컴파일 해주기 때문에 DEX는 하드웨어 플랫폼을 추가적으로 신경 쓸 필요가 없습니다. 즉 ART는 이 도구를 사용함으로써, 매개언어의 기본 철학인 이식성을 보장합니다.

### Binder IPC Proxy
IPC는 Inter Process Communication의 약자로 프로세스 간 신호(signal)와 데이터 교환을 가능케 하는 프레임워크 입니다. IPC는 정보 공유, 모듈화, 편의성, 권한 분리, 데이터 고립(isolation), 안정성, 컴퓨팅 연산 성능 개선 등의 기능을 제공합니다.

문서 초반에서 언급한 바와 같이 시스템 서비스도 사용자 응용 프로그램과 같은 계층(유저 공간)에서 실행되는 리눅스 프로세스 입니다. 프로세스는 기본적으로 isolation의 원칙을 지켜야 합니다. 만약 프로세스 간의 직접적인 통신이 가능하다면, 악의를 가지고 만든 프로세스 A가 프로세스 B를 죽일 수도 있습니다. 그러므로 프로세스 간 통신은 간접(indirect)적으로 이루어져야 합니다. 

일반적으로 유닉스 계열 운영체제에서 프로세스 간 통신을 지원하는 방법에는 Shared Memory와 Message Passing, Semaphore 등이 있습니다. 이를 System V 프로세스 통신이라고도 합니다. 안드로이드 라이브러리 언어인 libc(bionic)는 System V IPC를 지원하지 않습니다. 대신 Binder IPC를 지원합니다.

![Binder](/images/android/binder.png)

안드로이드는 RPC(Remote Procedure Call)의 원리를 이용하여 동일 시스템 내 프로세스 간 통신이 이루어지도록 합니다. 사실 RPC는 클라이언트와 서버가 서로 다른 시스템에 있음(분산 시스템 환경)에도 불구하고 클라이언트에서 마치 로컬의 루틴(메소드)를 호출하는 것처럼 보이게 하는 기술인데, 안드로이드는 이를 분산된 환경이 아니라 시스템 내 프로세스 통신으로 이용한 것입니다. RPC 기술을 통해 클라이언트에서 서버의 메소드를 호출하면 커널의 binder 프레임워크가 내부에서 이를 처리해 줍니다. 클라이언트는 서버 쪽 메소드를 호출할 때, 로컬 프로세스가 아닌 다른 프로세스의 메소드를 호출했다는 사실조차 모르도록 추상화되어 있습니다. 안드로이드에서는 AIDL(Android Interface Definition Language)로 클라이언트와 서버 프로그램 간에 소통할 수 있는 인터페이스 규약을 정의합니다.(윈도우가 MIDL을 지원하는 것과 같은 논리입니다.) AIDL을 작성하면 컴파일 시점에 자바 코드로 작성된 Proxy 혹은 Stub 파일이 자동으로 추출됩니다. 클라이언트에서 다른 프로세스의 특정 메소드를 호출하면 내부적으로 RPC가 Stub(메소드 구현체)을 호출합니다.

커널의 binder 드라이버(*/dev/binder*에 위치)는 Proxy 혹은 Stub 등과 같은 인터페이스 구현체가 전달한 파라미터를 마샬링(marshalling)합니다. 여기서 마샬링이란 자바(고수준 언어)로 작성된 자료구조를 데이터 전송(client to kernel binder)에 알맞는 형식으로 변환하는 것을 말합니다.

### 참고
**전문 서적**
- [Android System Programming - Porting, customizing, and debugging Android HAL](https://www.amazon.com/Android-System-Programming-customizing-debugging/dp/178712536X)
- [Inside the Android OS: Building, Customizing, Managing and Operating Android System Services (Android Deep Dive)](https://www.amazon.com/Inside-Android-OS-Customizing-Operating/dp/0134096347)
- [Embedded Android: Porting, Extending, and Customizing](https://www.amazon.com/Embedded-Android-Porting-Extending-Customizing/dp/1449308295)

**블로그**
- [Android Framework - 네이버](https://m.blog.naver.com/PostList.naver?blogId=bl2019&categoryName=an..%C2%A0framework&categoryNo=8&logCode=0)
- [안드로이드 어플리케이션이 실행되기까지 - 티스토리](https://sanseolab.tistory.com/32)
- [Android 프로세스의 통신 메커니즘: 바인더](https://d2.naver.com/helloworld/47656)

**문서**
- [Deep Dive into Android IPC/Binder Framework at Android Builders Summit 2013](https://androidbuilderssummit2013.sched.com/event/XXUyge/deep-dive-into-android-ipcbinder-framework-alekshellip)