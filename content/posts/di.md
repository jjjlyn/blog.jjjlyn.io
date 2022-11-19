---
title: "DI(Dependency Injection)란"
date: 2022-11-18T13:39:56+09:00
draft: true
tags:
- Android
---

# DI (Dependency Injection)

DI란 특정 클래스(객체) 안에서 필요한 객체를 생성·사용하는 대신, 외부에서 생성하여 이를 필요로 하는 클래스에 주입하여 사용하는 일종의 테크닉입니다. 객체지향의 SOLID 원칙 중 하나인 제어의 역전(DIP)을 구현한 방법 중 하나입니다. 구현체(Implementation)보다 추상체(Abstraction)에 의존하여 프로그래밍하는 것을 권장합니다.

## Dependency란?

A가 B에 의존한다는 것은 B가 변경되면 이에 따라 A도 변경될 수 있음을 의미합니다. 이는 유지보수 비용을 증가시키거나 예기치 못한 버그를 발생시킬 수 있습니다. 그러나 의존성이 없는 코드를 짜기란 불가능합니다.

의존성은 크게 4가지 타입으로 구분됩니다.

- Implementation Inheritance : 가장 강한 형태의 의존성 입니다. (`abstract class`나 `open class`를 implements한 형태)
- Composition : 객체(Object)의 호출(invocation)과 생성(creation)이 분리되지 않은 상태입니다. 전체에서 부분의 생성도 책임지기 때문에, 전체와 부분이 동일한 생명주기를 갖습니다.(전체가 종료될 때 부분도 파괴됩니다) 전체는 부분의 구현체를 압니다.
- Aggregation : 객체의 호출과 생성을 분리한 상태입니다. 더 이상 전체에서 부분의 생성을 책임지지 않습니다. 부분은 전체의 생성자를 통해 주입됩니다. 부분은 외부에서 생성된 것이므로, 전체와 부분의 생명주기가 다릅니다. 전체는 생성자를 통해 부분을 주입받기 때문에 부분의 실제 구현체(concrete class)를 알지 못합니다.
- Interface Inheritance : The Best Practice!! (`interface`를 implements한 형태) 이 경우 테스트하기 가장 좋은 코드가 됩니다.

## 의존성 주입 (DI)

객체 간 의존성을 줄이기 위해서는 객체의 호출과 생성을 분리하는 것이 좋습니다. 객체지향의 SOLID 원칙 중 단일 책임 원칙(Single Responsibility Principle)에서 권장하는 사항입니다. 앞서 언급한 바와 같이 DI란 특정 컴포넌트가 객체를 생성하여, 이를 필요로 하는 다른 객체에 주입하는 일련의 과정을 의미합니다. 예를 들어 안드로이드의 MainActivity에서 객체A를 생성한 후, 이를 객체B의 생성자에 전달한다고 가정해 봅시다. 이 경우 Activity 컴포넌트가 **객체B에 대한 객체A의 의존성 주입**을 담당하고 있는 것입니다. 보통의 경우 컴포넌트가 필요한 의존성을 직접 생성하여 주입하기보다는 DI 라이브러리가 이 역할을 대신하게 됩니다.

의존성 주입의 종류에는 크게 세 가지가 있습니다.

- 생성자 주입(Constructor injection)
  컴포넌트를 생성할 수 있는 권한이 주어질 때 사용할 수 있는 방법입니다.
- 변수 주입(Field injection)
  컴포넌트를 생성할 수 있는 권한이 없을 때 사용하는 방법입니다. 보통 안드로이드 프레임워크에서 생명주기를 갖는 컴포넌트는 개발자가 생성자를 호출하여 생성할 수 있는 권한이 없습니다.(Activity, BroadcastReceiver, ContentProvider, Service 등)
- 메서드 주입(Method injection)

Scope: different objects can have different lifecycles

Injector: 찾은(looked-up) 의존성 객체를 이를 필요로 하는 Field Property에 할당(assign)해 주는 것

생성된 객체에 대한 참조 정보를 갖고 있다가, 의존성을 요청하는 코드를 발견한 경우 lookup 작업을 수행한다.

## Dagger2

생명주기를 갖는 특정 **컴포넌트**(Component)에 **모듈**(Module)을 설치하여 사용합니다. 모듈은 **의존성**(Dependency)를 관리합니다.

컴포넌트는 기본적으로 Application 전체의  생명주기를 따르기 때문에, 이는 그보다 작은 범위에서 의존성이 필요한 경우에는 메모리 낭비가 일어날 수 있습니다. Dagger2는 **서브 컴포넌트**(SubComponent)를 통해 보다 작은 범위의 생명주기를 지원합니다(e.g. ActivityScope).

## Hilt

Dagger2가 안드로이드 전용이 아닌 자바 범용 라이브러리이다 보니, 안드로이드 개발 시 추가적으로 보일러플레이트 코드를 적용해야 하는 번거로움이 있긴 합니다. 이를 개선하기 위해 안드로이드 어플리케이션에 맞게 설계된 Hilt가 출시되었습니다.

Hilt는 Dagger2 라이브러리를 감싸는 Wapper입니다. Dagger2에서 사용하는 `@Component`, `@Subcomponent` 어노테이션을 제거하고 새로운 어노테이션을 사용합니다.

- Application 클래스에는 `@HiltAndroidApp`을 사용합니다.
- Activity, Fragment, Service, Bradcast Receiver, View 등에는 `@AndroidEntryPoint`를 사용합니다.
- ViewModel에는 `@HiltViewModel`을 사용합니다.

`@HiltAndroidApp`은 Dagger2의 `@Component`, `@AndroidEntryPoint`, `@HiltViewModel` 등은 `@SubComponent`로 보면 됩니다. Hilt에서는 Dagger2에서 했던 Field Injection 등을 위한 일련의 절차를 거칠 필요가 없습니다. Hilt에서 내부적으로 처리해줍니다.

Dagger2에서 Field Injection을 위해 거치는 개략적인 절차입니다.

```kt
@SubComponent
class ActivityComponent {
    fun inject(activity: Activity)
}

class MainActivity : AppCompatActivity {

    @Inject
    lateinit var classA: ClassA

    override fun onCreate(savedInstanceState: Bundle){
        application?.let {
            component.inject(this@MainActivity)
            classA.execute()
        }
    }
}
```

Hilt는 이런 과정이 필요 없습니다.
```kt
@AndroidEntryPoint // 대신 컴포넌트 생명주기에 맞는 annotation을 붙여야 합니다.
class MainActivity : AppCompatActivity {
    @Inject
    lateinit var classA: ClassA

    override fun onCreate(savedInstanceState: Bundle){
        classA.execute() // 바로 호출
    }
}
```

자바는 리플렉션(Reflection)이라는 툴을 제공합니다. 리플렉션이란 특정 클래스의 프로퍼티, 부모 클래스, 구현한 인터페이스, 메서드 등의 정보를 소스코드·바이트코드 혹은 메모리 파싱을 통해 런타임 환경에서 분석할 수 있는 도구입니다. 이는 런타임 환경에서 이루어지는 것이기 때문에 성능 저하의 요인이 될 수 있습니다. Square사는 annotation processor을 이용하여 컴파일 작업을 수행하기 이전에 코드 파싱이 이루어지도록 하는 획기적인 아이디어를 생각해 냈습니다.

### annotation processor란?
- 소스코드를 파싱하는 기능
- 소스 파일을 생성하는 기능


