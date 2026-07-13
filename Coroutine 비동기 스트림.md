# Coroutine Flow 펀더멘털


- Suspend
    - 일시중단 함수. 실행을 일시적으로 중단했다가 나중에 재개할 수 있는 suspension point를 제공
        - 단일 값 반환. 비동기 코드를 동기처럼 순차적으로 쓰고 읽게 하는 용도
        - 함수 실행 일시중단해도 스레드 블로킹 안함. 
            - 여러 값을 시간에 걸쳐 순차처리 하는 비동기 스트림의 기반이 됨
        - 콜백 지옥 대체.
            ```
            kotlin
            api.getUser { user -> 
                api.getPosts(user) { posts ->
                    api.... { ... ->
                        ...
                    }
                } 
            }
            ```
            이런 비동기코드를 동기 코드처럼 위 -> 아래로 읽게 하려고 등장
            비동기 코드를 동기 코드처럼 작성하니 가동성 극대화
        - suspenstion point 지점에서 딜레이가 발생할때 스레드 반납. 블로킹 아님.
        효율적인 자원 사용
        - 단, 값은 suspend fun 하나당 하나만 반환
        - 그리고 suspend은 Coroutine 블록이나 다른 Suspend fun 안에서만 호출 가능

    - 그럼 콜백을 무조건 안써야 하냐? 언제 쓰는게 적절하게 쓰는거냐?
        - 콜백 당연히 써야하지, 근데 비동기 관련 코드에는 거의 Suspend 함수 쓴다고 보면 됨
        - 단발성 비동기 결과 1번 받는 함수에 많이 씀 (api호출, DB조회, 파일 읽기 등)
        - Flow로 비동기 결과 여러번 받는 비동기 스트림 함수에도 많이 씀 (실시간 변화, 채팅, 이벤트 구독 등)

    - 비동기를 동기처럼 작성, 가독성 극대화 -> 단발성 비동기 코드 실무 예시
        - 보통 여러 비동기 코드를 조합하는 용도로 많이 사용
        ```
        kotlin
            fun placeOrder(userId: Ing, cardId: Long) {
                userApi.getUser(userId) { user ->
                    cardApi.getCard(user, cardId) { card ->
                        orderApi.createPlaceOrder(cardId, card.token) {
                            ...
                        }
                    }
                }
            }

            suspend fun placeOrder(userId: Int, cardId: Long): Order {
                val user = userApi.getUser(userId) // await 1
                val card = cardApi.getCard(user, cardId) // await 2, user 필요
                return orderApi.createPlaceOrder(cardId, card.token) // await 3, card 필요
            }

            콜백이었다면 3중 중첩 + 각 콜백에서 각각 에러처리 등등..
        ```

        ```
            suspend fun loadProductPage(id: Long): ProductPage = coroutineScope {
                val product = async { productApi.get(id) }
                val reviews = async { reviewApi.getByProduct(id) }
                val related = async { recommendApi.related(id) }

                ProductPage(
                    product = product.await(),
                    reviews = reviews.await(),
                    related = related.await(),
                )
            }

            순차로 짜면 하나에 300ms라 치면 900ms, 병렬로 하면 300ms
            실무에선 combine 많이 씀
        ```
    - 함수 실행 일시중단해도 스레드 블로킹 안한다?
        - Suspend는 suspendtion point를 제공할 뿐이지 (중단 가능한 함수) 쓴다고 무조건 중단되는건 아님
        - 중단되는 경우는 delay()나 NonBlocking IO 라이브러리 (예를들어 Retrofit)같이 중단을 지원하는 함수를 써서 실제 중단이 이루어졌을때만 스레드 중단을 함
        - 논블로킹 함수를 사용하지 않는다면 명시적으로 Dispatcher 선언하여 블로킹코드를 격리하는 방식으로 스레드 점유 해제 가능
        ```
        kotlin
            // 논블로킹 지원하지 않는 라이브러리 및 함수 미사용시 스레드 점유 및 블로킹
            suspend fun bedCase(): User {
                return jdbcTemplate.queryForObject(...)
            }

            // 블로킹 되지 않고 중단 및 스레드 넘겨야하는 코드는 명시적으로 Dispatcher 선언함으로 스레드 넘김. 블로킹 -> 논 블로킹으로 바뀐게 아니라 블로킹을 격리한 거 

            suspend fun goodCase() = withContext(Dispatcher.IO) {
               jdbcTemplate.queryForObject(...)
            }
        ```
        suspend 하나로 블로킹 코드를 논 블로킹으로 바꿔주지는 않는다

---
- Flow
    - 여러 값 + 비동기 조합의 구독 가능한 비동기 스트림, Flow<T>
    - 시간에 걸쳐 도착하는 여러개의 값을 비동기 방식으로 순차처리하는 스트림
    - 콜백 지옥, 리스너 수동 등록 / 해제, RxJava의 Observeable 개념을 Coroutine으로 대체
        - 각각의 Suspend 함수는 하나의 값만을 반환하기 때문에 여러 값을 시간에 걸쳐 못 줌.
        - 콜백은 중첩, 에러 분산, 취소 전파가 어려움
        - 값의 순차 방출 (emit)
        - 비동기 대기 (suspend)
        - 트리 형태로 구조화된 동시성으로 취소 자동 전파  
    - 디버깅 난이도 높음
    - Hot, Cold등 여러 개념 숙지 필요

    ||단일 값|여러 값|
    |-|-|-|
    |동기|T (그냥반환)|Seqeunce<T>|
    |비동기|Suspend fun|Flow<T>| <- 비동기 스트림

    - 그럼 언제 쓰는게 적절하게 쓰는거냐?
        ```
        1. 값이 하나면: 
            - 아무리 비동기여도 suspend fun 이면 됨. Flow 사용할 필요 없음
        2. 값이 여러개인데 CPU 계산만 하고 대기(네트워크, DB 접근, 딜레이 등)가 없으면: 
            - Sequence 사용. Flow는 suspend 컨텍스트를 요구하기 때문에 오히려 오버헤드
        3. 시간에 걸쳐 여러 값이 도착하고 그 사이사이 비동기 대기가 있다면: 
            - suspend fun ***(): Flow<T> 사용
        ```
    - 실무에서 Flow 사용 필요 조건
        ```
        1. 시간축에 따라 연쇄되는 값: 
            - DB테이블 조회
            - 위치 값 업데이트
            - Socket 통신
            - 검색어 입력 스트림
            - 한번 요청 -> 한번 응답이 아니라 구독하는 동안 계속 응답 나올때

        2. 연산자 파이프라인 필요할 때
            - map, filter, find, debounce, combine. flatMatLatest 등등 변환 합성 연산자
        
        3. 취소 전파 중요할 때
            - 기본적으로 구조화된 동시성위에서 동작하는 스트림이기 때문에 스코프 취소되면 구독도 자동 해제임
            - 하위 코루틴 스코프도 자동으로 취소됨
        
        4. Compose 기반에서 UI State 기반으로 UI 업데이트
            - StateFlow 혹은 SharedFlow 기반으로 ViewModel -> UI 단방향 스트림에 활용
        ```
    - Flow 과용 주의
        ```
        1. 일회성 요청, 응답
        2. 동기 컬렉션 변환
        3. 디버깅 난이도 상대적으로 높고, Hot, Cold 개념 필수기 때문에 여러값 + 비동기 + 구독 수명관리에 대한 이해 없이 쓰면 비용만 늘어남
        ```

- Hot, Cold
    - 스트림이 구독자를 위해 매번 새로 생기는가, 스트림은 원래 존재하고 여러 구독자가 나눠 보는가?

        ||Cold|Hot|
        |-|-|-|
        |대표 타입|Flow (flow{}, Room, Retrofit)|StateFlow, SharedFlow|
        |생산 시작 시점|Collect 하는 시점에만 블록 실행|Collect와 무관하게 이미 흐르고 있음|
        |구독자마다|Collect 시작하면 각차 처음부터 블록 새로 실행 (독립된 스트림)|Collect 시점에 상관없이 같은 스트림을 공유 (브로드캐스트)|
        |구독자 0이면|아무일도 안 일어남|스트림은 그냥 계속 존재함 (StateFlow는 마지막 값을 보유하고 있는 상태)|
        |비유|주문 즉시 조리|라디오 방송|
    - 자칫 범하는 실수
        - stateIn / shareIn 없이 Cold Flow를 UI에서 여러 번 collect 
            - 그때마다 API/쿼리가 중복 실행됨. "여러 구독자가 결과를 공유해야 한다"면 Hot으로 변환해야 함.
        - SharingStarted.WhileSubscribed(5_000) 
            - 화면 회전(구독 잠깐 끊김) 때 상류 재시작을 막는 표준값.
        - StateFlow에 이벤트를 담지 말 것
            - 최신값만 유지하고 중복 값은 conflate라서 이벤트 연속 발생시 유실됨. 이벤트는 SharedFlow(Replay = 0)혹은 Channel 기반으로 

- Shared Flow, State Flow
    - 둘다 Hot Flow는 같고, 둘의 경계를 나누는 것은 "값을 그저 흘려보내는 역할만 하는 순수 스트림인가? 항상 최신값을 들고 있는 State Holder역할까지 같이 하는가?"에 나뉜다
    - 방출하는 것 자체에 의의를 분다면 Shared Flow (Toast, Navigation 등등), 화면에 그려질 상태가 중요하다면 State Flow

        ||Shared Flow|State Flow|
        |-|-|-|
        |구분|값을 방출하는 이벤트 버스 역할|+ 마지막 값을 항숭 보유하고 있는 State Holder 역할|
        |초기값|없어도 됨|필수로 있어야함|
        |현재값 접근|값을 들고 있지 않으니 접근 할 수 없음|value 접근 가능|
        |replay|0, 1, n으로 설정 가능|마지막 최신값을 보유하기 때문에 항상 1|
        |중복값|중복이 있어도 그대로 다 방출|같은 값 연속 방출이면 skip, equals 비교해서 다른 인스턴스라면 방출|
        |늦게 구독한 쪽|replay 설정 만큼만. (0이면 못받음)|최신값을 즉시 받음|
        |정체성|일반적인 상위 개념|Shared Flow 상속 및 replay = 1, 중복 제거|

    - 자칫 범하는 실수
        - 상태가 아닌 이벤트를 State Flow로 방출
            - 최신값만 유지하고 중복 값은 conflate라서 이벤트 연속 발생시 유실됨. 이벤트는 SharedFlow(Replay = 0)혹은 Channel 기반으로 
        - 상태를 SharedFlow(replay=0)로 방출
            - 늦게 구독한 UI는 현재 상태 몰라서 빈 화면 됨.
        - StateFlow에 mutable 객체 재사용
            - list.add()후 같은 참조를 StateFlow에 넣으면 equals 동일 판정되어 방출 skip됨. 항상 .copy(), .toList()로 교체

---

- Channel
    - SharedFlow(replay=0)와 Channel 차이

---

- Sequence

- Coroutine Scope
    - Supervisor Scope