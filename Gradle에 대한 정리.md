# Gradle에 대한 정리

1. gradle은 JVM 생태계 기반(Java, Kotlin, Scala, Groovy) 기반의 표준 빌드 도구. 일반 JVM뿐 아니라 C++ 웹까지 빌드함

```
root project
├── settings.gradle.kts      ← 프로젝트(루트) 정의: 이름, 어떤 모듈이 포함되는지
├── build.gradle.kts         ← 빌드 스크립트: 플러그인/의존성/컴파일 설정 (핵심)
├── gradle/
│   └── wrapper/             ← Gradle "버전 고정" 장치
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew / gradlew.bat    ← wrapper 실행 스크립트 (mac/리눅스 / 윈도우)
└── src/main/kotlin/
    └── ***.kt          ← 실제 코드 (관례적 경로)
```

2. Android에서 왜 기본 빌드도구로 채택했냐. Kotlin First와 마찬가지로 Gradle도 Kotlin으로 쓸 수 있기 때문. (.kts -> kotlin script) 예전에는 Groovy build.gradle 이었는데 요즘에는 Kotlin DSL build.gradle.kts를 쓰고, 자동완성, 타입체크까지 IDE에서 처리해줌

3. build.gradle.kts 안의 내용을보면 plugin, repositories, dependencies, kotlin, application등 많은 설정을 스크립트 작성 할 수 있음.

4. gradle의 핵심 역할은 스크립트로 작성된 각 설정은 task graph로 묶어 어떤 설정 작업을 우선 실행할것인지 결정함. 
    - 의존성 -> 컴파일 -> 테스트 -> 패키징

5. gradle wrapper인 gralew는 gradle 설치 안해도 ./gradlew 실행하면 gradle에서 제시하는 버전을 자동으로 받아서 씀. 그래서 팀원 모두가 똑같은 gradle버전을 쓸 수 있음
    - 보통 gradle build가 아니라 ./gradlew build로 reademe에서 안내함

6. android와 순수 JWM이 다른점
    - 진입점
        - android: Activity, Manifest
        - 순수 JVM: main() 함수
    - 산출물
        - APK, AAB
        - JAR 또는 ./gradlew run
    - 빌드 도구
        - 둘다 Gradle