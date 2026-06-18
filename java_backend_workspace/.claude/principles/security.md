# 보안 (Iron Rule)

- **시크릿이나 자격증명을 절대 하드코딩하지 않는다.** JWT secret, DB 비밀번호, Redis 패스워드, 외부 API 키 등은 `application-*.yml` 의 `${ENV:default}` 로만 주입하고, 운영 값은 배포 파이프라인의 환경변수/Secrets 로 공급한다.
- `SecurityConfig` 의 `permitAll` 목록은 **명시된 경로만 공개**다. 새 엔드포인트를 공개하려면 이 목록을 직접 수정해야 하고, PR 본문에 공개 사유를 남긴다. 기본은 항상 인증 요구.
- 사용자 제어 데이터에 대해 **입력 검증**(Bean Validation + `@Valid`)과 **출력 인코딩**(Jackson 기본 이스케이프) 을 적용한다. 원시 문자열을 쿼리/JPQL 에 직접 concat 하지 않는다 (QueryDSL 또는 바인딩 파라미터 사용).
- 통합 키와 외부 서비스에 **최소 권한 원칙**을 적용한다. DB 계정, 외부 API key 는 읽기/쓰기 분리 가능하면 분리한다.
- 인증(auth)과 인가(authorization)를 **암묵적 동작이 아닌 명시적 인수 기준**으로 처리한다. 메서드 보안은 `@Secured`, `@PreAuthorize` 로 선언하고, 파라미터에는 `@LoginAccount` / `SecurityUtils.getLoginId()` 로 주체를 꺼내 쓴다.
- 인증/인가 경로(`SecurityConfig`, JWT 필터, `RefreshTokenService`) 를 수정할 때는 **해당 변경에 대한 테스트를 반드시 추가**한다. 공개 경로 추가, 필터 순서 변경, 토큰 TTL 조정은 보안 영향 범위가 크므로 독립 커밋으로 분리한다.
- **민감 정보 로깅 금지.** 토큰·비밀번호·OTP·검증 코드는 마스킹을 거쳐야 하고, 디버그 레벨이라도 평문으로 남기지 않는다. SQL 로그는 `local`/`development` 프로필에서만 활성화한다.
- 암호화/복호화는 공통 유틸(`CryptoUtil` 등)을 경유한다. 서비스 내부에서 `MessageDigest`·`Cipher` 를 직접 다루지 않는다.
- webhook 등 공개 경로는 `permitAll` 이지만 **서명/토큰 검증**을 엔드포인트 진입 직후 수행한다. 공격자가 임의 payload 를 주입할 수 있다는 전제로 작성한다.
- OWASP Top 10 중 **SQL Injection / Broken Access Control / Sensitive Data Exposure / Security Misconfiguration** 네 카테고리를 PR 체크리스트로 항상 확인한다.
