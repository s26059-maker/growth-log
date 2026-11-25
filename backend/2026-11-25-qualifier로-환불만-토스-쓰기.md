# @Qualifier로 환불만 토스 쓰기

> 📅 2026-11-25 · 백엔드 기초 · 75점
> `#Spring` `#Qualifier` `#Primary` `#빈`
> @Primary로 기본 결제를 정해둔 상태에서 @Qualifier로 환불만 토스를 쓰게 하고, 빈 이름 규칙과 우선순위를 정리

## 상황

결제는 `@Primary`를 붙인 **카카오페이**가 기본이야. 그런데 **환불만 토스**로 해야 해.

```java
@Primary
@Service
public class KakaoPayService implements PaymentService { /* 카카오페이 */ }

@Service
public class TossPayService implements PaymentService { /* 토스 */ }
```

그냥 `PaymentService`를 달라고 하면 기본값인 카카오가 들어와.

## @Qualifier

"이 자리엔 **이 이름표가 붙은 빈**을 넣어줘"라고 콕 집어 말하는 어노테이션이야.
평소엔 카카오 직원이 오는데, 환불 창구에만 토스 직원을 **지명**하는 거야.

```java
@Service
public class RefundService {

    private final PaymentService paymentService;

    public RefundService(@Qualifier("tossPayService") PaymentService paymentService) {
        this.paymentService = paymentService; // 여기만 토스
    }
}
```

## 붙이는 위치

| 위치 | 뜻 |
|---|---|
| 클래스 위 | 그 클래스 **자기 자신**한테 이름표를 붙임 |
| 생성자 매개변수 앞 | 여기에 **들어올 빈**을 고름 ✅ |

**중요한 점**: 들어올 빈을 고르는 거니까, 넣어주는 자리인 **생성자 매개변수 앞**에 붙여.

## 빈 이름 규칙

스프링은 클래스 이름에서 **첫 글자만 소문자**로 바꿔서 빈 이름을 자동으로 지어줘. 나머지는 그대로야.

| 클래스 | 빈 이름 |
|---|---|
| `KakaoPayService` | `kakaoPayService` |
| `TossPayService` | `tossPayService` |

이름은 **대소문자까지 정확히** 맞아야 해. 하나라도 틀리면 `NoSuchBeanDefinitionException`이 나면서 앱이 안 켜져. 그래도 앱 켤 때 바로 알 수 있으니 다행이야.

## @Primary와 @Qualifier

| | `@Primary` | `@Qualifier` |
|---|---|---|
| 어디에 | 구현 클래스 위 | 주입받는 자리 (생성자 매개변수 앞) |
| 뜻 | 대부분의 자리에 들어갈 **기본값** | **특정 자리**만 콕 집어 지정 |
| 예시 | 결제는 카카오 | 환불은 토스 |

**중요한 점**: 둘이 겹치면 **`@Qualifier`가 이겨.** 콕 집어 말한 게 우선이야.

## 실무에서는

- 문자열 이름은 오타가 나기 쉬워. 그래서 `@Service("toss")`처럼 빈 이름을 직접 짧게 정해두기도 해
- `@Query`랑 헷갈리지 말기. `@Query`는 JPA에서 SQL(JPQL)을 직접 적을 때 쓰는 거야

## 내가 놓쳤던 점

- 처음에 `@Qualifier`를 `@Query`랑 헷갈림
- `@Qualifier`를 **클래스 위**에 붙임. 들어올 빈을 고르려면 **생성자 매개변수 앞**
- 빈 이름은 첫 글자만 소문자, 나머지는 그대로: `tossPayService`
