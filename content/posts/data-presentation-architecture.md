---
title: "Data Presentation Architecture"
date: 2022-11-18T13:39:56+09:00
draft: true
categories:
- Android
tags:
- Android
---

# Android Data Presentation Architecture - MVC, MVP, MVVM, MVI

MVC, MVP, MVVM, MVI 등 `MV` 접두어가 달린 친구들은 **Data Presentation(Clean Architecture 관점에서 Presentation Layer)을 위한 소프트웨어 아키텍처 패턴**입니다.(**안드로이드 앱 전체를 위한 아키텍처로 착각하면 안 됩니다!!**) GUI 어플리케이션은 어찌되었든 사용자에게 fancy~~한 화면을 보여주는게 주 목적입니다. [오늘도 개발자가 안 된다고 말했다](http://www.yes24.com/Product/Goods/97919905)라는 책을 아시나요? UI 변경은 하루가 멀다하고 일어나는 이슈입니다. 그렇기 때문에 외형만 바꾸되 그 기반을 이루는 데이터에 변형이 일어나지 않도록 하는 것이 리소스(시간, 비용) 측면에서 매우 중요합니다. 예를 들어 UI 변경 부분, UI 변경 외(外) 부분 등 소스코드의 역할을 나누면 유지보수가 쉬워지겠죠. 더불어 UI 관련 외 부분에 대한 유닛 테스트(Unit test)도 가능해질 것입니다. 즉 `Model-View-Whatever` 소프트웨어 아키텍처 패턴은 관심사 분리(Separation of Concern)와 테스트하기 쉬운 코드 작성(Testable Code)을 위한 것입니다.

## Model And View가 정확히 의미하는 바는?

**Model**

우리가 유닛 테스트 할 수 있는 것에는 무엇이 있을까?
의심스러운 디자인 테크닉. 기술적 한계는 코드가 device 혹은 emulator 기반으로 실행된다는 것이다. 우리는 개발 환경에서 테스트하고 싶잖아. 그래서 androidTest와 test 디렉토리가 나뉜거다. 이론적으로 유닛 테스트를 작성해서 에뮬레이터 환경에서 돌릴 순 있지만 안정성도 떨어지고 시간도 오래 걸린다. 클라우드(Firebase Test Lab) 에뮬 환경에서 돌린다고 하면 이건 돈이 많이 든다. 현실적으로 로컬 유닛 테스트를 많이 해봐야 한다는건데, 이걸 해결하기 위해 우리는 안드로이드 컴포넌트와 자바 컴포넌트를 분리한 것이다. 

ConcreteData로부터 field1만 읽다가 field1 + field2의 결과값을 보여주려면 어떻게 해야하는가?
MainActivity에서 concatenation을 하거나 ConcreteData 클래스에 결합된 문자열이 반환되는 메소드를 추가할 수 있을 것이다. 그러나 둘 다 단점이 있다. MainActivity에서 문자열 결합이 이루어지면 이걸 유닛 테스트하기가 상당히 어려워진다. 만약 ConcreteData 내부에 메소드를 추가한다고 하면, UI를 업데이트 하는 역할만 하는 ConcreteData에 다른 역할을 부여(비지니스 로직?)하는 것이기 때문에 이 또한 문제가 된다. 만약 미래에, 네트워크 통신을 다른 팀에 의지하고 있으면 어떻게 할 것인가? 

**View**

## 비지니스 로직이란 무엇인가?

CRUD 아님. UI 업데이트 로직 아님. 

## MVC (Model-View-Controller)


## MVP (Model-View-Presenter)

![MVP](/images/android/mvp.png)

비록 실무에서 사용하지는 않았으나 아키텍처 패턴의 변천기를 아는 것은 여전히 중요하기에 간단히 다룹니다.

안드로이드 초창기에는 MVC와 유사한 아키텍처 패턴을 주로 사용했습니다. 안드로이드의 MVC에서는 Activity 혹은 Fragment가 Controller와 View의 역할 모두를 수행하며, Model은 데이터를 
Activity를 최대한 많이 만들어서 다른 앱들과 통신하기 위함. 이러한 이유로 액티비티는 인스턴스화 할 수 없고 오로지 인텐트를 사용하여 시작할 수 있음. 액티비티를 직접적으로 인스턴스화 하지 못하기 때문에 액티비티 생성자에 특정 의존성을 주입할 수가 없다. 또한 액티비티는 생명주기를 갖고 있어 이를 모두 상속받아야 한다. 그리하여 액티비티는 특별한 테스트 도구가 없는 한 유닛 테스팅이 어려워 실제 기기나 AVD 에뮬레이터를 통해 통합 테스트에 의존하는 수밖에 없다. 특별한 테스트 도구(e.g. Roboletric)나 통합 테스트를 사용하는 것은 속도가 느리기도 하거니와, firebase test lab과 같은 테스팅 클라우드를 이용하는 경우 비용적인 측면에서도 부담이 있다.
이러한 문제를 해결하기 위해 액티비티와 차후 나타난 프래그먼트를 위한 Humble Object 패턴이 도입된다. 이는 액티비티의 복잡다단한 로직을 최대한 추출해서 분리하는 패턴이다. 그 중 유명한 방법이 MVP 아키텍처 패턴이다. 액티비티와 뷰계층(android.widget.View)은 View가 되고, presenter는 모델로부터 데이터를 가져와 비지니스 로직을 수행하며, 모델은 mvc의 m의 역할을 수행한다. 즉 어플리케이션의 데이터를 관리하는 역할을 한다. Presenter는 데이터를 가공하여 View에 업데이트한다. View 또한 user interaction이 발생했을 때 Present를 호출한다. 두 컴포넌트의 관계로 인해 둘 간의 관계(contract)를 인터페이스로 정의해야 한다.

```kt
interface Presenter {
    fun loadUsers()

    fun validateInput(text: String)
}

interface View {
    fun showUsers(users: List<User>)

    fun showInputError(error: String)
}
```

뷰와 유즈케이스(모델)을 Presenter에서 모두 관리한다. PresenterImpl은 View와 GetUsersUseCase 객체에 의존성을 갖는다. GetUsersUseCase를 통해 사용자 목록을 받은 후 View의 showUsers()가 호출된다. 

```kt
class PresenterImpl(
    private val view: View,
    private val getUsersUseCase: GetUsersUseCase
): Presenter {

    private val scope = CoroutineScope(Dispatchers.Main)

    override fun loadUsers(){
        scope.launch {
            getUsersUseCase.execute().collect { users ->
                view.showUsers(users)
            }
        }
    }

    override fun validateInput(text: String){
        if(text.isEmpty()){
            view.showInputError("Invalid input")
        }
    }
}
```

View 구현체는 이런 식으로 표현된다. 
```kt
class MainActivity : ComponentActivity(), View {
    @Inject
    private lateinit val presenter: Presenter
    private lateinit val usersAdapter: UsersAdapter
    private lateinint val editText: EditText
    private lateinit val errorView: TextView

    override fun onCreate(savedInstanceState: Bundle?){
        super.onCreate(savedInstanceState)

        editText.addTextChangedListener(object : 
            TextWatcher {
                override fun afterTextChanded(s: Editable?){
                    presenter.validateInput(s?.toString().orEmpty())
                }
            }
        )

        presenter.loadUsers()
    }

    override fun showUsers(users:List<User>){
        usersAdapter.add(users)
    }

    override fun showInputError(error: String){
        errorView.text = error
    }
}
```

Presenter가 백그라운드 작업 수행을 종료할 때 Context 누수가 발생할 수 있다. 액티비티 생명주기에 Presenter 객체의 메모리를 release하는 작업이 필요하다.

```kt
interface Presenter {
    fun close()
}
```

```kt
override fun onDestroy(){
    presenter.close()
    super.onDestroy()
}
```

```kt
class PresenterImpl(
    private val view: View,
    private val getUsersUseCase: GetUsersUseCase
): Presenter {
    private val scope = CoroutineScope(Dispatchers.Main)

    override fun close(){
        scope.cancel()
    }
}
```

Flow 객체에 대한 구독을 취소했으니 Activity가 destroy되었을 때 더이상 업데이트를 받지 않게 된다.

비록 이 MVC를 개선한 MVP가 초기에는 많이 쓰였고 아직도 많은 상용앱에서 사용하고 있지만, 이 또한 여러 문제를 야기한다. MVVM 패턴을 장착한 AAC 출시 이후, 추가적으로 Jetpack Compose까지 더해서 더 좋은 data flow를 제공한다.

## MVVM (Model-View-ViewModel)

![MVVI](/images/android/mvvi.png)
AAC(Android Architecture Components)를 통해 뷰 업데이트 로직과 비지니스 로직을 분리합니다.

MVVM은 액티비티나 프래그먼트로부터 로직을 추출하는 Humble Object 패턴과는 다른 접근방식이다. View는 액티비티 혹은 프래그먼트, Model은 data management, ViewModel은 View가 요청할 때 Model로부터 데이터를 요청합니다. 세 컴포넌트는 단방향의 흐름을 갖습니다. View는 ViewModel에 의존하고, ViewModel은 Model에 의존합니다. 많은 View가 동일한 ViewModel을 사용할 수 있기 때문에 유연합니다. View에서 데이터를 업데이트 하기 위해 ViewModel은 Observer 패턴으로 구현해야 합니다. ViewModel은 Observable을 이용하여, View가 이를 구독하고 데이터를 실시간으로 변경할 수 있도록 해야 합니다. 

AAC 라이브러리는 ViewModel 클래스를 제공하는데, 이는 액티비티와 프래그먼트의 생명주기에 맞춰 데이터 흐름을 관리합니다. Coroutine extensions와 결합된 AAC ViewModel은 액티비티나 프래그먼트가 더이상 데이터를 받으면 안되는 생명주기 상태에 있을 때 Flow 혹은 Coroutine의 구독을 중지하여 컨텍스트 누수를 막습니다.

Clean Architecture 관점에서 ViewModel은 UseCase로부터 데이터를 가져온 후 entity를 Framework 계층이 필요로 하는 객체로 변환하는 역할을 합니다(데이터 가공처리). 또한 사용자가 넘긴 데이터를 entity로 가공하여 UseCase로 넘기는 반대의 역할도 수행합니다. 

```kt
class MyViewModel(
    private val getUsersUseCase: GetUsersUseCase
): ViewModel(){

    private val _usersFlow = MutableStateFlow<List<UiUser>>(listOf<UiUser>())
    val usersFlow: StateFlow<List<UiUser>> = _usersFlow

    fun load(){
        viewModelScope.launch {
            getUsersUseCase.execute().map {
                // Convert List<User> to List<UiUser>
            }.collect {
                _usersFlow.value = it
            }
        }
    }
}
```

## MVI (Model-View-Intent)

> 주의: MVI의 Intent는 안드로이드 컴포넌트 통신에 사용되는 Intent가 아닙니다.

MVVM과 대비되는 것이 아니라, MVVM 위에 Intent 기능을 얹힌 아키텍처 입니다.

## 참고

**전문 서적**

- [Clean Android Architecture: Take a layered approach to writing clean, testable, and decoupled Android applications](https://www.amazon.com/Clean-Android-Architecture-decoupled-applications/dp/180323458X)