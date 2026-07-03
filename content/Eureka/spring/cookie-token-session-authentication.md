---
title: "쿠키, 세션, 토큰 인증 — Spring에서 로그인 상태는 어떻게 유지되는가"
date: 2026-07-03
tags: [Java, Spring, Spring Security, Cookie, Session, JWT, Authentication]
---

# 쿠키, 세션, 토큰 인증 — Spring에서 로그인 상태는 어떻게 유지되는가

HTTP는 기본적으로 **무상태(Stateless)** 프로토콜입니다. 요청이 끝나면 서버는 "누가 요청했는지"를 기억하지 못합니다. 그런데 우리는 로그인 한 번으로 여러 페이지를 옮겨 다녀도 로그인 상태가 유지되죠. 이 글에서는 그 상태 유지를 가능하게 하는 세 가지 방식 — **쿠키, 세션, 토큰(JWT)** — 이 각각 무엇이고 어떻게 다른지, 그리고 이전 글에서 다룬 **Filter Chain의 어디쯤**에서 처리되는지 정리합니다.

---

## 1. 쿠키(Cookie) — 모든 것의 물리적 매개체

쿠키는 서버가 브라우저에게 "이 값을 저장해두고, 앞으로 나한테 요청할 때마다 같이 보내줘"라고 지시하는 메커니즘입니다.

```
[서버 → 브라우저]  Set-Cookie: JSESSIONID=A1B2C3; Path=/; HttpOnly; Secure; SameSite=Lax
[브라우저 → 서버]  Cookie: JSESSIONID=A1B2C3
```

| 속성 | 의미 |
|---|---|
| `HttpOnly` | JS(`document.cookie`)로 접근 불가 → XSS로 탈취 방지 |
| `Secure` | HTTPS에서만 전송 |
| `SameSite=Strict/Lax/None` | 다른 사이트發 요청에 쿠키를 실어 보낼지 제어 → CSRF 완화 |
| `Max-Age` / `Expires` | 만료 시점 |

**중요**: 쿠키 자체는 "인증 방식"이 아니라 "값을 담아 옮기는 그릇"입니다. 이 그릇 안에 세션 ID를 담으면 세션 인증, JWT를 담으면 토큰 인증이 되는 것입니다.

---

## 2. 세션(Session) 기반 인증

### 2.1 동작 원리

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Server
    participant Store as Session Store (메모리/Redis)

    B->>S: POST /login (id/pw)
    S->>S: 인증 성공
    S->>Store: 세션 생성, 세션에 사용자 정보 저장 (JSESSIONID: A1B2C3)
    S-->>B: Set-Cookie: JSESSIONID=A1B2C3

    B->>S: GET /mypage (Cookie: JSESSIONID=A1B2C3)
    S->>Store: JSESSIONID로 세션 조회
    Store-->>S: 사용자 정보 반환
    S-->>B: 200 OK (마이페이지 데이터)
```

- 서버는 `JSESSIONID`라는 랜덤 식별자만 쿠키로 내려주고, **실제 사용자 정보(누구인지, 권한이 뭔지)는 서버 메모리(또는 Redis 등 외부 저장소)에 보관**합니다.
- 클라이언트는 그저 "내 세션 ID는 이거예요"라고 매 요청마다 제시할 뿐, 자기 정보를 들고 다니지 않습니다.

### 2.2 Spring에서의 위치

세션 인증은 Servlet 표준(`HttpSession`) 자체 기능이라 Spring Security를 쓰지 않아도 동작하지만, Spring Security를 쓰면 `SecurityContext`를 `HttpSession`에 저장하는 방식(`HttpSessionSecurityContextRepository`)으로 통합됩니다. 이전 글에서 설명한 **Filter Chain 중 `SecurityContextPersistenceFilter`(또는 최신 버전의 `SecurityContextHolderFilter`)** 단계에서, 요청이 들어올 때마다 세션에서 `SecurityContext`를 꺼내 `SecurityContextHolder`에 채워 넣습니다.

### 2.3 장단점

| 장점 | 단점 |
|---|---|
| 서버가 세션을 강제로 만료/삭제 가능 (즉시 로그아웃 처리 쉬움) | 서버가 상태(세션)를 들고 있어야 함 → 스케일아웃 시 세션 클러스터링/Redis 필요 |
| 클라이언트에 민감 정보 노출 없음 (ID만 전달) | 모바일 앱, 여러 도메인 간 API 등에서 쿠키 기반이라 다루기 불편할 수 있음 |

---

## 3. 토큰(Token) 기반 인증 — JWT

### 3.1 JWT 구조

JWT(JSON Web Token)는 `.`으로 구분된 3부분으로 구성됩니다.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSIsInJvbGUiOiJVU0VSIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└────── Header ──────┘└──────────── Payload ─────────────┘└──────────── Signature ────────────┘
```

- **Header**: 알고리즘 정보 (`{"alg": "HS256", "typ": "JWT"}`)
- **Payload**: 클레임(claims) — 사용자 정보, 권한, 만료시간 등 (`{"sub": "user1", "role": "USER", "exp": 1735689600}`)
- **Signature**: `Header + Payload`를 서버의 비밀키(또는 개인키)로 서명 → **위변조 방지**

> Header/Payload는 Base64Url 인코딩일 뿐 **암호화가 아니므로** 디코딩하면 누구나 내용을 볼 수 있습니다. 비밀번호처럼 민감한 정보를 Payload에 넣으면 안 됩니다. Signature 덕분에 "내용을 볼 수는 있어도 조작은 못하는" 구조입니다.

### 3.2 동작 원리

```mermaid
sequenceDiagram
    participant B as Browser/App
    participant S as Server (Resource Server)

    B->>S: POST /login (id/pw)
    S->>S: 인증 성공, JWT 생성 (서버 비밀키로 서명)
    S-->>B: { accessToken: "eyJ...", refreshToken: "eyJ..." }

    B->>S: GET /mypage  (Header: Authorization: Bearer eyJ...)
    S->>S: 서명 검증 (비밀키로 재계산 후 비교) — DB/세션 조회 없음!
    S-->>B: 200 OK (마이페이지 데이터)
```

핵심 차이: **서버는 아무 상태도 저장하지 않는다(Stateless).** 토큰 자체에 사용자 정보와 서명이 담겨 있으므로, 서버는 매 요청마다 서명만 검증하면 되고 세션 저장소를 조회할 필요가 없습니다.

### 3.3 Access Token / Refresh Token 분리

토큰은 탈취되면 만료 전까지 세션처럼 강제 종료가 어렵기 때문에(Stateless라 서버가 "무효화" 리스트를 안 두는 이상 취소 불가), 보통 두 개로 나눠 씁니다.

| 토큰 | 수명 | 용도 |
|---|---|---|
| Access Token | 짧음 (예: 30분) | 매 API 요청의 인증에 사용 |
| Refresh Token | 김 (예: 2주) | Access Token 재발급 전용, DB/Redis에 저장해두고 검증 |

```
Access Token 만료 → 클라이언트가 Refresh Token으로 /reissue 요청
                  → 서버가 저장해둔 Refresh Token과 대조 후 새 Access Token 발급
```

### 3.4 Spring Security에서의 위치

JWT 인증은 Spring Security 표준 필터에 포함되어 있지 않아서, 보통 **커스텀 Filter**(`OncePerRequestFilter`를 상속)를 만들어 Filter Chain에 끼워 넣습니다.

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String token = resolveToken(req); // Authorization 헤더에서 추출
        if (token != null && jwtProvider.validateToken(token)) {
            Authentication auth = jwtProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth); // 세션 없이 매 요청 인증 정보 설정
        }
        chain.doFilter(req, res);
    }
}
```

이 필터는 보통 `UsernamePasswordAuthenticationFilter` 앞에 등록되어, **DispatcherServlet에 요청이 도달하기 전**(지난 글의 Step 2 지점)에 인증을 완료시킵니다.

---

## 4. 세션 vs 토큰 — 정면 비교

| 항목 | Session | Token (JWT) |
|---|---|---|
| 상태 저장 위치 | 서버 (메모리/Redis) | 클라이언트 (토큰 자체에 정보 포함) |
| 서버 확장성 | 세션 클러스터링/Sticky Session 필요 | Stateless라 수평 확장에 유리 |
| 즉시 로그아웃/강제 만료 | 세션 삭제로 즉시 가능 | 불가능에 가까움 (블랙리스트 운영 필요) |
| 매 요청 비용 | 세션 저장소 조회 필요 | 서명 검증만 (조회 없음, 단 Redis 캐시 병행하기도 함) |
| 주 사용처 | 전통적 서버 렌더링 웹, 단일 도메인 | REST API, 모바일 앱, MSA, 여러 도메인 간 인증(SSO 등) |
| 전달 매개체 | 쿠키(주로 `JSESSIONID`) | Authorization 헤더(`Bearer`) 또는 쿠키에 담아 전달 가능 |

> 참고로 토큰도 **쿠키**에 담아 전달할 수 있습니다 (`Set-Cookie: accessToken=eyJ...; HttpOnly`). "쿠키 vs 토큰"은 사실 잘못된 이분법이고, 진짜 비교는 "**세션 방식 vs 토큰 방식**"이며 쿠키는 둘 중 어느 쪽이든 실어 나르는 운반체일 뿐입니다.

---

## 5. 보안 체크리스트

- **XSS 방지**: 토큰을 `localStorage`에 저장하면 JS로 탈취 가능 → 가능하면 `HttpOnly` 쿠키에 담기
- **CSRF 방지**: 쿠키는 브라우저가 자동으로 실어 보내므로 CSRF에 취약 → `SameSite` 속성 + CSRF 토큰 병행 (참고로 순수 헤더 기반 JWT는 브라우저가 자동으로 붙여주지 않으므로 CSRF에 상대적으로 안전)
- **Refresh Token 탈취 대비**: 서버 DB에 저장해두고 재발급 시마다 검증 + 회전(Rotation) 전략 사용
- **토큰 만료 시간**: Access Token은 짧게, 탈취되어도 피해 최소화

---

## 6. 정리

- 쿠키는 "값을 담아 옮기는 그릇"일 뿐, 인증 방식 자체가 아니다.
- **세션 인증**은 서버가 상태(사용자 정보)를 들고 있고 클라이언트는 식별자(JSESSIONID)만 전달 — 즉시 무효화가 쉽지만 확장성에서 불리.
- **토큰(JWT) 인증**은 클라이언트가 서명된 정보를 직접 들고 다니고 서버는 검증만 함 — Stateless라 확장에 유리하지만 즉시 무효화가 어려움.
- 실무에서는 Access/Refresh Token을 나누고, 지난 글에서 다룬 **Filter Chain의 앞부분**(DispatcherServlet 도달 전)에서 커스텀 필터로 인증을 처리하는 구조가 일반적이다.
