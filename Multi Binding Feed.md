# Multi Binding Feed

유동적인 Feed를 구성할때 보통 서버에서 List<Feed>의 형태로 Feed들을 내려받아 앱에서 보여주곤 한다.

각 기능 역할별로 모듈이 나눠져있다는 가정하에, Feed를 구성하는 여러 Item들이 자연스럽게 Common에 모일수밖에 없는데 이때 여러 문제가 발생한다.

순진하게 그냥 짜게 되면 중앙(common module)쪽에서 
```
when (uiState) {
    is SearchBarUiState   -> SearchBarItem(...)
    is MyAccountUiState   -> MyAccountItem(...)
    is RecentAccountUiState -> RecentAccountItem(...)
    // ... 새 타입 추가할 때마다 이 when 을 계속 수정 (개방-폐쇄 위반)
}
```
이런식으로 계속 누적이 되게 된다. 이 when 은 모든 아이템 타입을 한 파일이 알아야 하고, 아이템이 늘 때마다 비대해진다. 

그 외에 다른 문제도 있는데, 각 기능, 역할별로 모듈이 이미 나눠져있는데 그 핵심이라 볼 수 있는 UI Composable은 정작 개별 모듈이 아닌 중앙 모듈에 있을 수 밖에 없게 되는 것이다.

책임이 분산되니 당연히 캡슐화는 안되고, 캡슐화가 안됨으로 인해 일어날 수 있는 추가 발생 문제가 생긴다.

---

멀티바인딩은 이 when 을 Map 조회로 바꾼다. 

각 UI Component는 자기 모듈에서 @IntoMap로 맵에 등록 하고, 
중앙 common 모듈의 코드는 map[uiState::class] 로 조회만 하고 렌더 요청만 한다.

실제 렌더는 각 피쳐 모듈에서 하는 방식.

이렇게 되면 새로운 ViewType -> UiState가 생길때마다 UI Composable, (remember)State, Event등이 같이 생기기 마련인데 이들을 common에 몰아 넣는게 아니라 
책임이 있는 각 모듈로 분산할 수 있고, 렌더도 책임 있는 각 모듈이 진행한다는 것이다.


```
중앙 when (결합 ↑)                  멀티바인딩 (결합 ↓)
   ┌────────────────────────┐      ┌─────────────────────────────┐
   │ when(uiState) {        │      │ map[uiState::class]          │
   │   is A -> ...          │      │                              │
   │   is B -> ...          │      │  A 모듈 ─@IntoMap(A)─┐        │
   │   is C -> ... ← 매번 수정│      │  B 모듈 ─@IntoMap(B)─┼─▶ Map  │
   │ }                      │      │  C 모듈 ─@IntoMap(C)─┘ (자동집계)│
   └────────────────────────┘      └─────────────────────────────┘
```
---

그 결과, 어느 모듈에서건 서버로부터 Feed를 받아오게 되고
1. Entity -> VO -> UiState순으로 매핑 연결
2. 각 모듈에서 ItemState 및 UI Composable 생성
3. @IntoMap 기반 Binding

조건을 채우면

1. A모듈에서 Feed ViewType List 받아옴
```
{
    [
        viewType: A-A
        ...
    ],
     [
        viewType: B-A
        ...
    ],
     [
        viewType: B-B
        ...
    ],
     [
        viewType: C-A
        ...
    ],
}
```

2. A 모듈에서 LazyColumn 기반으로 Feed 구현

3. B-A, B-B, C-A ViewType은 A모듈에 정의되어있는 ViewType은 아니지만 Multi Binding을 통해 B모듈, C모듈에서 렌더링 및 상호작용을 직접 함

이런식으로 흐름이 이어지게 된다