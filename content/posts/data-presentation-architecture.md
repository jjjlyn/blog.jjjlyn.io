---
title: "Model-View-Whatever"
date: 2022-11-23T13:39:56+09:00
draft: false
tags:
- Android
---

> MV-xxx 패턴 진짜 안 헷갈리시나요? 3년차 개발자인 저는 부끄럽지만 아직도 제대로 이해하고 쓰는 건지 모르겠습니다. MVC, MVP, MVVM, 여기서 발전한 MVI까지...
> 
> 최근 모스타트업 기술면접에서 MVVM 아키텍처를 설명해보라는 질문이 들어왔습니다. 분명 지겹도록 이를 적용해 왔고 문서로 따로 정리를 하기도 했었죠. 그러나 머리로는 이해를 했으나 가슴으로 받아들이지 못한 것인지 MVP와 비교하기 시작하면 몹시 헷갈리더라고요. 왜 MVVM이 *unidirectional*이고 MVP가 *bidirectional*인지 제대로 설명했어야 했는데 면접 중 갑자기 '그런데 왜 MVVM이 단방향 흐름이지?' 라고 스스로 의문이 들어 자가당착에 빠졌었네요.

<figure style="display:block; text-align:center;">
  <img src="/images/naneun-solo-youngcheol-gaesori.jpg">
  <figcaption style="text-align:center; font-size:15px; color:#808080">
    아.. 영철 아저씨의 말을 새겨들어야 하거늘 ㅠㅠ 이해는 가슴으로 합시다
  </figcaption>
</figure>

MV-xxx 친구들은 View(Activity, Fragment 등)가 비대(~~뚱뚱~~)해지지 않도록 관리하는 차원에서 나온 아키텍처 입니다. 착각하면 안 되는게 **안드로이드 앱 전체를 위한 아키텍처가 아닙니다.** View를 관리하기 위한 아키텍처 입니다. Clean Architecture에 의거하면 Data Presentation Layer를 위한 아키텍처라고도 할 수 있겠습니다.

View는 기본적으로 테스트하기 난감합니다. UI를 직접 업데이트하는 기능을 포함하고 있어 플랫폼 종속적이기 때문입니다. 이 말은 즉슨 UI를 업데이트하는 기능만 View에 남겨두고, 플랫폼에 종속적이지 않은 다른 기능은 다른 클래스로 빼버리면 그 클래스는 테스트가 용이할 수 있다는 의미입니다.

## MVC (Model-View-Controller)
MVC는 간단하게 언급만 하고 넘어가겠습니다. 여기서 Model은 간단히 말해 화면에 표시할 데이터라고 보면 됩니다. View는 사용자에게 보일 화면입니다(레이아웃과 Activity/Fragment 등). Controller는 Model과 View를 묶는 역할을 합니다. View에 Model을 표시합니다. View인 `R.layout.activity_main`은 `MainActivity`에 표시됩니다. 이제 inflate된 View의 자식인 텍스트뷰(이 역시 View)에 특정 데이터(Model)를 표시하는 역할(`setText()`)을 Controller가 담당합니다. 이로써 Controller에 의해 Model과 View가 묶이게 됩니다. 안드로이드에서는 이 MVC 패턴을 적용해봐야 코드 개선이 안됩니다. 테스트하기 어렵습니다. 왜냐하면 **Activity/Fragment가 View와 Controller의 역할 모두를 수행하기 때문입니다.** 이 말인 즉슨 View와 Controller의 경계선이 모호하다는 말입니다. Activity/Fragment가 필연적으로 비대해질 수밖에 없는 구조입니다.

## MVP (Model-View-Presenter)

![MVP](/images/android/mvp.png)

안드로이드 초창기(Activity만 있던 시절)에는 MVC와 유사한 아키텍처 패턴을 주로 사용했습니다. 그러나 Activity는 인스턴스화 할 수 없고 오로지 Intent를 사용하여 시작할 수 있습니다. Activity를 직접적으로 인스턴스화 하지 못하기 때문에 Activity constructor에 특정 의존성을 주입할 수 없습니다. 또한 Activity는 생명주기를 갖고 있어 이를 모두 상속받아야 합니다. 그리하여 특별한 테스트 도구가 없는 한 유닛 테스팅이 어려워 실제 기기나 AVD 에뮬레이터를 통해 통합 테스트에 의존할 수밖에 없게 됩니다. 특별한 Test Tool(e.g. Roboletric)이나 통합 테스트를 사용하는 것은 속도가 느리기도 하거니와, Firebase Test Lab과 같은 Testing Cloud를 이용하는 경우 비용적인 측면에서도 부담이 있기도 합니다.
이러한 문제를 해결하기 위해 Activity와 추후 나타난 Fragment를 위한 Humble Object 패턴이 도입됩니다. 이는 Activity의 View + Controller 기능이 얽히고 섥힌 복잡다단한 로직을 최대한 추출해서 분리하는 패턴입니다. 그 중 유명한 방법이 MVP 아키텍처 패턴입니다. Model과 View는 MVC의 MV와 같습니다. 

달라진 점은 Presenter입니다. 아래 코드는 Presenter를 interface로 정의한 것입니다.

```kt
interface Presenter {
    fun loadUsers()

    fun validateInput(text: String)
}
```

Presenter는 기존 Controller과는 다릅니다. Controller는 앞에서 언급한 바와 같이 `view.setText("blah blah")` 플랫폼 메서드를 직접 호출하여 화면에 데이터를 띄우는 역할을 합니다. 그러나 Presenter는 이 역할을 직접 하지 않고 **Activity/Fragment에 위임**합니다. 이 MVP 구조에서는, Activity/Fragment가 사용자 이벤트를 Presenter에 알립니다. Presenter는 View로부터 받은 사용자 이벤트를 처리합니다. 필요에 따라 Model에 View에 보내줄 필요할 데이터를 요청할 수도 있고, 필요한 데이터가 없으면 Presenter가 알아서 처리할 수도 있습니다. 예를 들어 Activity에서 EditText에 addTextChangedListener 콜백을 등록하여, 사용자 입력이 일어날 때마다 Presenter에 이벤트를 보낸다고 합시다(Presenter의 `validateInput()`을 호출한다고 합시다). Presenter는 View로부터 Input에 대한 정보를 받아서 이를 처리합니다. 어떻게 처리할까요? 아래 코드는 View interface를 정의한 것입니다.

```kt
interface View {
    fun showUsers(users: List<User>)

    fun showInputError(error: String)
}
```

Presenter는 `validateInput()` 메서드 내부에서 View의 `showInputError()`을 호출하여, UI 처리를 Activity/Fragment에 위임합니다. 즉 Controller처럼 View와 Model을 직접적으로 연결해주는게 아니라, View가 알아서 UI 부분은 처리하도록 그 역할을 넘기는 것입니다.

Presenter가 View의 이벤트를 받아 Model로부터 필요할 때 데이터를 받고 이벤트 발생에 대해 View에 적합한 처리(UI 업데이트)를 하도록 시키려면 Presenter는 View와 Model의 Reference를 모두 갖고 있어야 하겠죠.

아래 코드는 Presenter의 구현체입니다.

```kt
class PresenterImpl(
    private val view: View, // View에 대한 Reference
    private val getUsersUseCase: GetUsersUseCase // Model에 대한 Reference
): Presenter {

    private val scope = CoroutineScope(Dispatchers.Main)

    override fun loadUsers(){
        scope.launch {
            getUsersUseCase.execute().collect { users ->
                // View에 Model로부터 받은 데이터를 제공해주는 동시에 특정 UI 처리를 하도록 간접적으로 지시(?)합니다.
                // 단 Presenter는 플랫폼과 독립되어 있으므로 UI 업데이트를 직접 관장하지 않습니다.
                view.showUsers(users)
            }
        }
    }

    // View로부터 받은 이벤트를 처리합니다.
    override fun validateInput(text: String){
        // 이 경우 Model로부터 데이터를 받을 필요 없습니다.
        // Presenter가 View에 특정 UI 처리를 하도록 간접적으로 지시합니다.
        if(text.isEmpty()){
            view.showInputError("Invalid input")
        }
    }
}
```

다음은 View interface를 상속받은 MainActivity 입니다.

```kt
class MainActivity : ComponentActivity(), View {
    @Inject
    lateinit val presenter: Presenter
    
    private lateinit val usersAdapter: UsersAdapter
    private lateinint val editText: EditText
    private lateinit val errorView: TextView

    override fun onCreate(savedInstanceState: Bundle?){
        super.onCreate(savedInstanceState)

        editText.addTextChangedListener(object : 
            TextWatcher {
                override fun afterTextChanded(s: Editable?){
                    // View에서 일어난 Input 이벤트를 Presenter에 알립니다.
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
        // MainActivity는 플랫폼 의존적입니다. 
        // 직접 UI를 업데이트하는 기능을 합니다.
        errorView.text = error
    }
}
```

## MVVM (Model-View-ViewModel)

![MVVI](/images/android/mvvi.png)

MVVM의 ViewModel은 AAC(Android Architecture Component) ViewModel과 무관합니다. MVVM 패턴은 마이크로소프트가 고안한 것이며, 
Complex maintenance issues can arise as apps are modified Complex maintentance issues can arise as apps are modified and grow in size and scope. These issues inclde the tight coupling between the UI controls and the business logic, which increases the cost of making UI modifications, and the difficulty of unit testing such code. 
Separate the application's business and presentation logig cform its user interface. Data Binding and Commands -> ViewModel updates the model'

MVVM은 Activity나 Fragment로부터 로직을 추출하는 Humble Object 패턴과는 다른 접근방식입니다. Model, View, ViewModel 세 컴포넌트는 **의존성 측면에서 단방향의 흐름을 갖습니다.** View는 ViewModel에 의존하고, ViewModel은 Model에 의존합니다. 

<details>
<summary>여기서 단방향이란</summary>
<div markdown="1">

단방향의 흐름을 갖는다는 의미가 갑자기 혼란을 주었던 이유는, View가 이벤트를 ViewModel에 전달하면 ViewModel이 View에 데이터를 제공해주는 것이 아닌가하는 의문이 들었기 때문입니다. 일단 여기서 ViewModel이 View에 데이터를 제공하는 것부터 틀린 말입니다. View는 ViewModel에 의존성을 갖고, ViewModel의 변수를 구독(관찰)하는 것이지, ViewModel이 View에 데이터를 직접 제공하거나 하지는 않습니다.

</div>
</details>

MVVM은 

## MVI (Model-View-Intent)

> 주의: MVI의 Intent는 안드로이드 컴포넌트 통신에 사용되는 Intent가 아닙니다.

MVVM과 대비되는 것이 아니라, MVVM 위에 Intent 기능을 얹힌 아키텍처 입니다.

## 참고

**전문 서적**

- [Clean Android Architecture: Take a layered approach to writing clean, testable, and decoupled Android applications](https://www.amazon.com/Clean-Android-Architecture-decoupled-applications/dp/180323458X)