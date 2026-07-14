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

- 

### Qualifier

### @IntoMap, @IntoSet, @MapKey

### @EntryPoint

### @AssistedInject