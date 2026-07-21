# Hilt Annotation

Hilt, 혹은 더 넓게 DI왜 쓰는지는 누구든지 안다. 그리고 Hilt가 제일 확장성 좋고 제공하는 기능 많고 읽기 쉬운 것 같다.

근데 인간적으로 어노테이션 좀 많은 것 같다. 머릿속에 정리좀 해야할 것 같음

|Annotation|상황|
|-|-|
|@Inject|작성한 클래스 제공|
|@Binds|인터페이스 -> @Inject 구현체 연결|
|@Provides|생성자 @Inject 못 붙히는 외부 라이브러리 반환, 혹은 생성에 로직 필요한 경우|
|@AndroidEntryPoint + 필드 @Inject|Activity, Fragment에서 사용|
|@HiltViewModel|ViewModel에서 사용|
|@Qualifier|같은 타입 여러개일때 구별|
|@IntoMap, @IntoSet|여러 군데 등록 -> Map, Set으로 자동 집계|
|@Singleton|앱 전역 1개 재사용, 그 외 여러 Scope있음|
|@EntryPoint|Hilt 안 붙는 클래스에서 접근|
|@AssistedInject|생성자 일부는 컴파일시 Hilt가 주입 + 일부는 런타임시 직접 주입 혼합|
|@Module, @InstallIn|@Binds, @Provides담는 그릇|
|@HiltAndroidApp|Application클래스의 붙히는 Hilt 의존성 트리 시작점|

### 그래프
```kotlin
@HiltAndroidApp (Application)                     ← 그래프 시작
        │
@AndroidEntryPoint (Activity/Fragment/…)          ← 주입 활성화
        │  필드 @Inject
        ▼
   [ 의존성 그래프 ]
        ├ 내 클래스        → 생성자 @Inject
        ├ 인터페이스        → @Module + @Binds
        ├ 외부 라이브러리    → @Module + @Provides
        ├ 같은타입 구분      → @Qualifier / @Named
        ├ 여러개 집계        → @IntoMap / @IntoSet
        ├ ViewModel        → @HiltViewModel
        └ Hilt 밖 접근      → @EntryPoint
        │
   각 노드의 수명 = @InstallIn(Component) + Scope(@Singleton 등)
```

---

### @AndroidEntryPoint

- Activity, Fragment, View, Service, BroadcastReceiver에 붙히는데, 해당 컴포넌트에 멤버 주입 가능한 컨테이너 달아줌.

```kotlin
@AndroidEntryPoint
class ***Activity: AppCompatActivity() {
    @Inject lateinit var logger: EventLogger
}
```
- Activity, Fragment의 경우 Android프레임 워크가 생성자를 직접 호출하므로 Hilt가 생성자 주입 불가함. 예외적으로 필드 주입

|상황|필요|
|-|-|
|내가 만든 클래스 (생성자 접근 가능)|생성자 @Inject|
|Android 컴포넌트 (프레임워크가 생성)|@AndroidEntryPoint + 필드 @Inject|
|인터페이스 / @Inject 못 붙히는 외부 라이브러리|Module 필요|

### @Module, @InstallIn, @Binds, @Provides

- 생성자 주입이 안되는 것들
    - 인터페이스, 외부라이브러리 객체(retrofit, room, okHttp등)은 @Inject 생성자 못 달음. 그래서 Module에서 객체 생성 방법을 알려줘야함

- @Module은 객체 생성 밥법 제공 모음
- @InstallIn은 어느 컴포넌트 스코프에 엮을지 지정
- @Provides는 외부 라이브러리등 생성 로직을 직접 써야할 때 선언
    ```kotlin
    @Module
    @InstallIn(SingletonComponent::class)
    object NetworkModule {
        @Provides
        fun provideRetrofit(): Retrofit = 
            Retrofit.Builder()
                ...
                .build()
    }
    ```
- @Binds는 interface <-> impl 연결. 몸통없이 매핑만 담당
    ```kotlin
    @Module
    @InstallIn(SingletonComponent::class)
    abstract class RepoModule {
        @Binds
        abstract fun bind***Repository(
            impl: ***RepositoryImpl
        ): ***Repository
    }
    ```

### Qualifier

- 같은 타입을 여러개 제공해야할 때도 있는데, 이때 Hilt가 어느걸 제공해야할지 모른다. 이때 이름표 붙히는거랑 동일
- @Named 도 있는데 문자열 기반으로 이름 붙히는거라서 @Qualifier가 안전
```kotlin
@Qualifier 
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

---

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    fun io() = Dispatchers.IO
}

---

class worker @Inject constructor(
    @IoDispatcher private val dispatcher: CoroutineDispatcher
)
```

### @IntoMap, @IntoSet, @MapKey
- 여러 곳 (모듈)에서 각자 등록된걸 하나의 Map/Set으로 자동 집계.
    - 중앙의 거대한 When 제거. 등록 책임을 실제 책임이 있는 곳으로 분산
- @IntoSet
    ```kotlin
    @IntoSet
    @Module 
    @InstallIn(SingletonComponent::class)
    abstract class InterceptorModule {
        @Binds 
        @IntoSet
        abstract fun authInterceptor(impl: AuthInterceptor): Interceptor
    }
    // 주입: Set<@JvmSuppressWildcards Interceptor>

    ```
- @IntoMap + @MapKey — 키로 조회하는 맵. 키 타입은 커스텀 @MapKey로 정의.
    ```kotlin
    @MapKey
    annotation class ViewTypeKey(val value: KClass<out UiState>)

    @Module 
    @InstallIn(SingletonComponent::class)
    abstract class FeedItemModule {
        @Binds 
        @IntoMap
        @ViewTypeKey(SearchBarUiState::class)
        abstract fun searchBar(impl: SearchBarItemRenderer): FeedItemRenderer
    }

    // 중앙: when 없이 조회만
    class FeedRenderer @Inject constructor(
        private val renderers: Map<Class<out UiState>, @JvmSuppressWildcards FeedItemRenderer>
    ) {
        fun render(state: UiState) = renderers[state::class.java]?.render(state)
    }

    중앙 when (결합 ↑)                  멀티바인딩 (결합 ↓)
    is A -> ...                       A 모듈 ─@IntoMap(A)─┐
    is B -> ...                       B 모듈 ─@IntoMap(B)─┼─▶ Map (자동집계)
    is C -> ... ← 매번 수정             C 모듈 ─@IntoMap(C)─┘

    // @JvmSuppressWildcards: 코틀린 제네릭이 ? extends 와일드카드로 바뀌며 Dagger가 타입을 못 찾는 문제 방지용. 멀티바인딩 컬렉션 주입 시 관용적으로 붙임.
    ```

### @EntryPoint

- @AndroidEntryPoint를 못 붙히는 클래스 (라이브러리 클래스, ContentProvier, WorkManager 초기 진입 등)에서 그래프 의존성 꺼내올때

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface AnalyticsEntryPoint {
    fun logger(): EventLogger
}

val logger = EntryPointAccessors
    .fromApplication(context, AnalyticsEntryPoint::class.java)
    .logger()
```
- 이름이 @AndroidEntryPoint와 비슷하나 역할 다름. 
    - @AndroidEntryPoint = 컴포넌트에 주입 활성화
    - @EntryPoint = 그래프에서 수동으로 꺼내오는 창구.

### @AssistedInject

- 일부는 Hilt가 주입, 일부는 런타임에 인위적으로 넘겨야 할 때
```kotlin
class ArticleLoader @AssistedInject constructor(
    private val repository: ArticleRepository,   // Hilt 주입
    @Assisted private val articleId: Long,        // 런타임 전달
) {
    @AssistedFactory
    interface Factory {
        fun create(articleId: Long): ArticleLoader
    }
}

---

class SomeVm @Inject constructor(factory: ArticleLoader.Factory) {
    val loader = factory.create(articleId = 42)
}
```