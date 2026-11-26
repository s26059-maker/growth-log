# 회원 조회 API 리뷰: 서비스 층, Optional, DTO

> 📅 2026-11-26 · 코드 리뷰 · 85점
> `#코드리뷰` `#JPA` `#DTO` `#Optional` `#레이어드`
> 컨트롤러가 엔티티를 그대로 돌려주는 회원 조회 API에서 서비스 층 누락, Optional.get(), 비밀번호 노출을 찾아 DTO로 고친 리뷰

## 리뷰한 코드

id로 회원 한 명을 조회하는 API.

```java
@Entity
public class User {

    @Id @GeneratedValue
    private Long id;

    private String name;
    private String email;
    private String password;
}

@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;              // 문제 1

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {              // 문제 3
        return userRepository.findById(id).get();             // 문제 2
    }
}
```

## 서비스 층이 없음

컨트롤러가 **서비스를 건너뛰고** 레포지토리를 바로 써. 주문 받는 직원이 요리사를 건너뛰고 창고에 직접 들어가는 셈이야.
나중에 "탈퇴한 회원은 조회 안 되게" 같은 규칙이 생기면 넣을 곳이 없어.
→ `UserService`를 두고, 컨트롤러는 서비스만 부르게

## Optional을 확인 없이 꺼냄

`findById()`는 회원을 **`Optional`이라는 상자**에 담아서 줘. 있을 수도, 없을 수도 있다는 뜻이야.
`.get()`은 상자를 확인 안 하고 꺼내서, 없는 id면 `NoSuchElementException`이 터지고 사용자는 알 수 없는 500 에러를 받아.

```java
// before
userRepository.findById(id).get();

// after: 없으면 던질 에러를 직접 정함
userRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("회원이 없어요: " + id));
```

## 엔티티를 그대로 응답함

엔티티를 그대로 돌려주면 **모든 필드**가 JSON으로 나가.

```json
{ "id": 1, "name": "홍길동", "email": "hong@test.com", "password": "qwer1234" }
```

- **비밀번호가 그대로 노출**돼
- 엔티티에 필드를 추가하면 **API 응답까지 같이 바뀌어**

→ 보여줄 것만 담은 **응답 전용 DTO**를 만들기

```java
public record UserResponse(Long id, String name, String email) { // 비밀번호는 안 담음
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
    }
}
```

## 고친 코드

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public UserResponse getUser(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("회원이 없어요: " + id));
        return UserResponse.from(user);
    }
}

@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

## 내가 단 리뷰 코멘트

> **21번 줄**: 층을 나눠서 서비스 층을 만드는 게 좋아 보이는데 어떻게 생각하시나요? 나중에 회원 관련 규칙이 생기면 넣을 곳이 필요할 것 같아요.
>
> **24번 줄**: 비밀번호가 노출될 위험이 있어 보여요. 응답 전용 DTO를 만드는 게 어떨까요? 엔티티가 바뀔 때 API 응답이 같이 바뀌는 것도 막을 수 있어요.
>
> **25번 줄**: 없는 id면 에러가 날 수 있으니까 `orElseThrow`로 예외 처리를 해두는 거 어때요?

**중요한 점**: 리뷰는 명령보다 **제안 + 이유 + 질문**으로 쓰면 받는 사람이 방어적이 되지 않고 대화가 돼.

## 내가 놓쳤던 점

- 처음엔 문제를 하나도 못 찾아서 밑바닥부터 다시 배움
- `findById()`가 **Optional**을 돌려준다는 것, `.get()`이 위험하다는 것
- 엔티티를 그대로 돌려주면 **비밀번호까지** 나간다는 것
