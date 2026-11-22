---
name: backend-test
description: 백엔드 테스트 코드를 작성합니다. Service 테스트, Controller 테스트 작성이 필요할 때 사용하세요.
---

# Backend 테스트 작성

JUnit 5 + Mockito-Kotlin 조합으로 테스트 코드를 작성합니다.

## Instructions

### 1. 라이브러리 (고정)
- **JUnit 5**: `@Test`, `@BeforeEach`, `assertEquals`, `assertThrows<T>`
- **Mockito-Kotlin**: `mock<T>()`, `given().willReturn()`, `verify()`

### 2. 테스트 구조
```kotlin
class SomeServiceTest {
    private val repository = mock<SomeRepository>()
    private lateinit var service: SomeService

    @BeforeEach
    fun setUp() {
        service = SomeService(repository)
    }

    @Test
    fun `한글로 테스트 설명 작성`() {
        // given
        given(repository.findById(1L)).willReturn(Optional.of(data))

        // when
        val result = service.method(1L)

        // then
        assertEquals(expected, result)
        verify(repository).findById(1L)
    }
}
```

### 3. 규칙
- 메서드명: 한글 백틱 사용
- 구조: Given-When-Then 패턴
- Mock 생성: `mock<T>()` 함수 (어노테이션 금지)
- Stubbing: `given().willReturn()` 패턴
- 예외 테스트: `assertThrows<ExceptionType> { }`

## 예시

```kotlin
@Test
fun `사용자가 존재하지 않으면 NotFoundException을 던진다`() {
    // given
    given(userRepository.findById(1L)).willReturn(Optional.empty())

    // when & then
    assertThrows<NotFoundException> {
        userService.getUserById(1L)
    }
}
```

## 테스트 범위

- **Service 테스트**: 비즈니스 로직을 검증한다.
- **Controller 테스트**: API의 권한(`@Auth`·`@GroupAuth`로 누가 허용·거부되는지)을 검증한다. 권한은 컨트롤러 메서드의 선언에 있으므로 Service 테스트에서 권한 판정을 직접 호출해 흉내 내지 않는다.
  - `@WebMvcTest`, `MockMvc`, `@EnableAspectJAutoProxy`를 사용하고, 검사할 Aspect(`AuthAspect`, `GroupAuthAspect`)와 판정 서비스를 `@Import`로 등록한다.
  - API마다 선언한 권한이 정책과 맞는지 허용·거부 사례로 확인한다. 공개 범위별 세부 판정처럼 판정기 자체의 규칙은 판정기 테스트에서 검증한다.

## 역할 범위

- **O**: Service 테스트 작성, Controller 권한 테스트 작성
- **X**: 프로덕션 코드 작성 (별도 skill 사용)
