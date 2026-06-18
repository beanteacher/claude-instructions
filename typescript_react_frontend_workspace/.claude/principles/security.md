# 보안 (Iron Rule)

- **시크릿이나 자격증명을 절대 하드코딩하지 않는다.** API base URL 등 환경별로 달라지는 값은 `NEXT_PUBLIC_*` 환경변수로만 주입하고, 실제 시크릿(API key, 내부 서비스 URL 등)은 서버 사이드 환경변수로만 공급한다.
- **`dangerouslySetInnerHTML` 사용을 원칙적으로 금지한다.** 마크다운·HTML 렌더링이 필요한 경우 DOMPurify 등 sanitizer 를 반드시 통과시킨다. `app/layout.tsx` 의 테마 감지 스크립트처럼 외부 입력이 없는 정적 인라인 스크립트만 예외로 허용한다.
- **액세스 토큰은 메모리에만 보관한다.** `authStore.ts` 의 `accessToken` 은 zustand 인메모리 상태다. `localStorage`·`sessionStorage` 에 토큰을 저장하면 XSS 시 즉시 탈취된다.
- **인증 가드(`auth-guard.tsx`) 가 보호 라우트의 단일 진입점이다.** `PUBLIC_ROUTES` 에 등록되지 않은 모든 경로는 `AuthGuard` 를 통과해야 한다. 개별 페이지 컴포넌트에서 인증 여부를 중복 체크하지 않는다.
- **민감 정보 로깅 금지.** `accessToken`, 비밀번호, 개인정보는 `console.log` / `console.info` 에 출력하지 않는다. 디버그 목적이라도 평문으로 남기지 않는다. 로거는 `common/utils/code-utils.ts` 의 `logger` 를 사용하고, 프로덕션에서는 `logger.info` 이상 레벨만 출력한다.
- **외부 링크에는 `rel="noopener noreferrer"` 를 필수로 붙인다.** `target="_blank"` 와 함께 사용하지 않으면 탭나이핑(tabnapping) 공격이 가능하다.
- **`NEXT_PUBLIC_*` 환경변수는 클라이언트 번들에 노출된다.** 백엔드 호출 base URL (`NEXT_PUBLIC_API_BASE_URL`) 처럼 공개해도 무방한 값만 이 접두사를 사용한다. API key, 내부 시크릿, DB 접속 정보는 절대 `NEXT_PUBLIC_*` 로 선언하지 않는다.
- **의존성 보안 취약점을 주기적으로 점검한다.** `npm audit` 를 PR 병합 전 실행하고, high 이상 취약점은 즉시 패치 또는 사유 기재 후 억제한다. renovate / dependabot 도입을 권고한다.

---

## XSS (Cross-Site Scripting) 방어

### dangerouslySetInnerHTML

```typescript
// ❌ 금지 — 사용자 입력이 포함될 수 있는 곳
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// ❌ 금지 — 마크다운 렌더링 결과를 sanitize 없이 삽입
<div dangerouslySetInnerHTML={{ __html: marked(markdown) }} />

// ✅ 허용 — 외부 입력이 없는 정적 스크립트 (app/layout.tsx 의 테마 감지 스크립트 수준)
<script dangerouslySetInnerHTML={{ __html: `/* 정적 초기화 코드만 */` }} />

// ✅ 마크다운 렌더링 시 sanitize 필수
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(marked(markdown)) }} />
```

### iframe

```tsx
// ✅ sandbox 속성으로 권한 최소화
<iframe src={url} sandbox="allow-scripts allow-same-origin" />

// ❌ sandbox 없는 iframe
<iframe src={externalUrl} />
```

### 외부 링크

```tsx
// ✅ noopener noreferrer 필수
<a href={externalUrl} target="_blank" rel="noopener noreferrer">링크</a>

// ❌ rel 없는 새 탭 링크
<a href={externalUrl} target="_blank">링크</a>
```

---

## 토큰 보관 전략

| 토큰 종류 | 보관 위치 | 접근 방법 |
|----------|----------|----------|
| access token | zustand `authStore` (메모리) | `useAuthStore().getAccessToken()` |
| refresh token | httpOnly 쿠키 (백엔드가 Set-Cookie) | 직접 접근 불가 — `/auth/refresh` 호출로 갱신 |

### 왜 localStorage 금지인가

1. localStorage 는 동일 출처의 모든 JavaScript 에서 접근 가능하다.
2. XSS 취약점이 있으면 공격자가 `localStorage.getItem('accessToken')` 으로 즉시 탈취한다.
3. 메모리 저장 시 탭 전환·새로고침 시 소멸하지만, httpOnly 쿠키의 refresh token 으로 자동 복구된다 (`auth-guard.tsx` 마운트 시 `/auth/refresh` 호출).

```typescript
// ❌ 절대 금지
localStorage.setItem('accessToken', token.accessToken)
sessionStorage.setItem('token', JSON.stringify(token))

// ✅ 메모리(zustand)에만 저장
useAuthStore.getState().setToken(token)
```

---

## CSRF 방어

- refresh token 을 httpOnly + SameSite=Strict (또는 Lax) 쿠키로 저장하면 타 도메인에서 시작된 요청에 쿠키가 전송되지 않아 CSRF 가 방지된다.
- 백엔드가 CSRF 토큰을 요구하는 경우, 응답 헤더에서 추출해 axios 인스턴스의 request 인터셉터에서 자동 주입한다. `axios-interceptor.tsx` 에서 구현한다.
- 폼 제출은 `react-hook-form` 을 사용하고 `action` attribute 기반 전통적 폼 제출을 사용하지 않는다.

---

## 환경변수 보안

```
# .env.local (커밋 금지 — .gitignore 확인)
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080    # 공개 가능 — 클라이언트 번들 포함
INTERNAL_SECRET_KEY=...                           # 서버사이드 전용 — NEXT_PUBLIC 없음
```

- `.env.local`, `.env.production` 은 절대 커밋하지 않는다. `.gitignore` 에 이미 포함되어야 한다.
- `.env.example` 파일에 키 목록만 노출하고 실제 값은 빈 값 또는 예시 값으로 채운다.
- `NEXT_PUBLIC_*` 는 빌드 시 번들에 인라인되므로, 브라우저 개발자 도구로 누구나 확인 가능하다. 이를 전제하고 값을 결정한다.

---

## next.config.ts 보안 설정

```typescript
// next.config.ts
const nextConfig = {
  images: {
    // 허용된 도메인만 화이트리스트에 추가
    remotePatterns: [
      { protocol: 'https', hostname: 'your-storage.example.com' },
    ],
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          // CSP 는 인라인 스크립트가 있는 경우 nonce 전략과 함께 구성
          // { key: 'Content-Security-Policy', value: "default-src 'self'; ..." },
        ],
      },
    ]
  },
}
```

- `images.domains` (deprecated) 대신 `images.remotePatterns` 을 사용한다.
- CSP 헤더 추가 시, `app/layout.tsx` 의 테마 감지 인라인 스크립트에 nonce 를 적용해야 한다.

---

## 인증 가드 (`components/auth-guard.tsx`)

```typescript
// PUBLIC_ROUTES 에 없는 모든 경로는 AuthGuard 통과 필수
export const PUBLIC_ROUTES = ["/login"]  // common/constants.ts

// app/layout.tsx → AppLayout → AuthGuard → children
```

- 새 공개 라우트가 생기면 `common/constants.ts` 의 `PUBLIC_ROUTES` 에 추가한다.
- 개별 페이지 컴포넌트 (`page/*.tsx`) 에서 `useAuthStore().isAuthenticated()` 를 중복 체크하지 않는다. 가드에서 이미 보장된다.
- `AuthGuard` 는 마운트 시 `accessToken` 이 없으면 `/auth/refresh` 를 시도한다. refresh 실패 시 `/login` 으로 리다이렉트한다.

---

## OWASP Top 10 PR 체크리스트

PR 에 보안 관련 변경이 있을 때 아래 항목을 확인한다.

| 카테고리 | 확인 항목 |
|---------|---------|
| XSS | `dangerouslySetInnerHTML` 사용 여부, sanitizer 적용 여부 |
| 민감 데이터 노출 | `NEXT_PUBLIC_*` 에 시크릿 포함 여부, 로그 출력 여부 |
| 잘못된 인증 | `PUBLIC_ROUTES` 변경 시 사유 명시, `auth-guard.tsx` 우회 여부 |
| 보안 설정 오류 | `next.config.ts` 의 `remotePatterns` 허용 범위, CSP 헤더 |
| 취약한 의존성 | `npm audit` 결과 high 이상 취약점 없음 |
