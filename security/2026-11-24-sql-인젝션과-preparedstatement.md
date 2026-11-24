# SQL 인젝션과 PreparedStatement

> 📅 2026-11-24 · 보안 · 75점
> `#보안` `#SQL인젝션` `#PreparedStatement` `#JDBC`
> 문자열로 이어 붙인 로그인 SQL이 admin'-- 한 줄로 뚫리는 과정과 PreparedStatement 바인딩으로 막는 법을 정리

## SQL 인젝션

사용자가 입력한 값이 **SQL 코드의 일부가 돼버리는** 공격이야.
SQL을 문자열 더하기로 만들면, 입력값에 `'`나 `--` 같은 SQL 문법을 섞어서 쿼리 모양 자체를 바꿀 수 있어.

```java
public boolean login(String username, String password) throws SQLException {
    String sql = "SELECT * FROM users WHERE username = '" + username
               + "' AND password = '" + password + "'";

    ResultSet rs = connection.createStatement().executeQuery(sql);
    return rs.next(); // 결과가 한 줄이라도 있으면 로그인 성공
}
```

## 공격이 먹히는 과정

`users` 테이블

| id | username | password |
|---|---|---|
| 1 | admin | qwer1234 |
| 2 | sungwon | abcd5678 |

아이디 칸에 `admin'-- `, 비밀번호 칸에 `1234`를 넣으면 이렇게 만들어져.

```sql
SELECT * FROM users WHERE username = 'admin'-- ' AND password = '1234'
```

SQL에서 `--` 뒤는 **주석**이라 무시돼. DB가 실제로 실행하는 건 이거야.

```sql
SELECT * FROM users WHERE username = 'admin'
```

실행 결과

| id | username | password |
|---|---|---|
| 1 | admin | qwer1234 |

결과가 한 줄 나오니까 `rs.next()`가 `true` → 비밀번호를 몰라도 **admin으로 로그인 성공**.

**중요한 점**: 전체가 조회되는 게 아니라, 비밀번호 조건만 사라지고 **아이디 조건은 살아 있어.** 그래서 원하는 계정(관리자)으로 콕 집어 들어갈 수 있어.

## 글자를 막는 건 답이 아님

`'`나 `--`를 못 치게 막는 방식(블랙리스트)은 위험해.

- 우회할 길이 너무 많아. MySQL은 `#`도 주석이고, `' OR '1'='1` 같은 다른 공격도 있어
- 정상 사용자도 막혀. `O'Brien`처럼 이름에 `'`가 들어가는 사람이 있어

## PreparedStatement로 막기

SQL 모양을 먼저 정해두고, 값이 들어갈 자리는 `?`로 비워둬. 값은 나중에 **값으로만** 끼워 넣어. 이걸 **파라미터 바인딩**이라고 해.

```java
String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
PreparedStatement ps = connection.prepareStatement(sql); // SQL 모양을 먼저 확정
ps.setString(1, username); // 첫 번째 ?에 값 끼우기
ps.setString(2, password); // 두 번째 ?에 값 끼우기
ResultSet rs = ps.executeQuery();
```

**서류 양식**이라고 생각하면 돼. 양식(SQL 모양)은 미리 인쇄돼 있고, 빈칸(`?`)에는 **글자만** 적을 수 있어. 빈칸에 뭘 적어도 양식 자체는 못 바꿔.

같은 공격을 넣으면 DB는 이름이 **글자 그대로 `admin'-- `인 사람**을 찾아.

실행 결과

| id | username | password |
|---|---|---|
| (결과 없음) | | |

→ **로그인 실패**. 막혔어.

## 실무에서는

- JPA의 `findByUsername(username)` 같은 메서드나 `:username` 파라미터를 쓴 JPQL은 내부적으로 바인딩을 써서 안전해
- 하지만 `@Query`나 네이티브 쿼리에 **문자열을 이어 붙이면** JPA에서도 똑같이 뚫려
- 비밀번호를 `qwer1234`처럼 그대로 저장하는 것도 문제야. BCrypt 같은 **해시**로 바꿔서 저장해야 해

**중요한 점**: 막아주는 건 ORM이 아니라 **바인딩**이야. 어떤 도구를 쓰든 SQL에 입력값을 직접 이어 붙이지 마.

## 내가 놓쳤던 점

- 주석으로 사라지는 건 비밀번호 조건뿐이라, 전체가 아니라 **admin 한 줄**이 조회됨
- `'`, `--` 같은 글자를 막는 블랙리스트는 우회되고 정상 사용자도 막음 → **바인딩**이 정답
- ORM도 문자열을 이어 붙이면 뚫림. `?` 자리에 `setString`으로 값을 끼우는 코드 모양을 몰랐음
