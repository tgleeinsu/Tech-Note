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
    |동기|T|Seqeunce<T>|
    |비동기|Suspend fun|Flow<T>| <- 비동기 스트림

    - 그럼 언제 쓰는게 적절하게 쓰는거냐?
    ```
    ```


- State Flow
- Shared Flow





- Sequence
- Channel
- Coroutine Scope
    - Supervisor Scope