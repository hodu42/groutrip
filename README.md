# 🧳 그루트립 (Groutrip)

> 친구·가족 2~8인이 여행을 함께 계획하고, 결정하고, 정산하는 그룹 여행 협업 플랫폼.

| 기간 | 팀 구성 | 담당 |
|---|---|---|
| 2026.04 ~ 2026.06 | 2인 | **Backend · Frontend · 배포** |

SSAFY 15기 1학기 관통 프로젝트로 진행했습니다. 2인 팀이라 기획부터 배포까지 둘이 나눠 맡았습니다.

---

## ✨ 주요 기능

<!-- TODO: 기능별 스크린샷 또는 GIF. 실시간 동기화는 GIF 권장 -->

| 기능 | 설명 |
|---|---|
| 장소 검색 · 투표 | 가고 싶은 장소를 모아두고 그룹원이 투표로 결정 |
| 성향 설문 기반 추천 | 설문 결과를 반영해 일정에 맞는 장소 추천 |
| 이동 경로 · 비용 계산 | 확정된 장소 간 이동 경로와 예상 비용 산출 |
| 최소 송금 정산 | 나눠 낸 비용을 송금 횟수가 가장 적도록 정리 |
| 실시간 동기화 | 한 명이 바꾼 내용이 다른 그룹원 화면에 즉시 반영 (SSE) |

---

## 🛠️ 기술 스택

| 구분 | 사용 기술 |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS 4, Zustand, TanStack Query, React Router 7 |
| 실시간 · 시각화 | `@microsoft/fetch-event-source` (SSE), Recharts |
| Backend | Spring Boot 4, Spring Security · OAuth2 Client, JPA · QueryDSL, Spring AOP(AspectJ), JWT(jjwt) |
| DB · 마이그레이션 | PostgreSQL, Flyway |
| 문서화 · 테스트 | springdoc OpenAPI, JUnit |
| Infra | Docker Compose, AWS S3 (로컬 개발은 MinIO) |

---

## 👥 담당 영역

![시스템 구조도](%EB%8B%A4%EC%9D%B4%EC%96%B4%EA%B7%B8%EB%9E%A8/07_%EC%8B%9C%EC%8A%A4%ED%85%9C%EA%B5%AC%EC%A1%B0%EB%8F%84.png)

<details>
<summary>ER 다이어그램</summary>

![ER 다이어그램](%EB%8B%A4%EC%9D%B4%EC%96%B4%EA%B7%B8%EB%9E%A8/04_ER%EB%8B%A4%EC%9D%B4%EC%96%B4%EA%B7%B8%EB%9E%A8.png)

</details>

2인 팀에서 **기반 · 비용 · 인프라** 영역을 맡았습니다.

| 영역 | 담당 |
|---|---|
| 인증 · 그룹 관리 · 정산 · 실시간 동기화 · 배포 | **김민준** |
| 설문 · 추천 · 투표 · 장소 검색 | **정희성** |

---

## 🙋 담당 구현

### 소셜 로그인과 토큰 관리

- Google · Kakao OAuth2 로그인
- Access Token 30분 / Refresh Token 7일, Refresh는 **HttpOnly 쿠키**에 저장해 스크립트 접근 차단
- axios 인터셉터에서 만료를 감지해 재발급을 자동 처리. 화면 코드가 토큰을 다루지 않도록 분리

### 계좌 정보 암호화

수취 계좌 정보는 유출 시 피해가 큰 데이터라 **AES-GCM 컬럼 단위 암호화**를 적용했습니다.

### 테스트

직접 눈으로 확인하기 어려운 로직을 중심으로 작성했습니다.

- 테스트 클래스 **31개** / `@Test` **104건**
- `SettlementCalculatorTest`, `SseServiceTest`, `GroupPermissionAspectTest`, `OAuthLoginServiceTest` 등

---

## 🧯 트러블슈팅

### 1. 그룹 권한 검사가 API마다 흩어짐

**문제** — 그룹 관련 API가 늘어날수록 "이 사용자가 이 그룹의 멤버인가"를 확인하는 코드가 컨트롤러마다 반복됐습니다. 한 곳이라도 빠뜨리면 권한 구멍이 생깁니다.

**해결** — `GroupPermissionAspect`로 검증을 AOP 한 곳에 모으고, 각 API는 어노테이션만 붙이도록 했습니다. 새 API를 추가할 때 검증을 빠뜨릴 여지가 줄었습니다.

### 2. 정산 시 송금 관계가 지나치게 복잡해짐

**문제** — 여행 중 비용을 번갈아 내다 보면 정산 시점에 주고받을 관계가 얽힙니다. 모든 채권·채무를 1:1로 처리하면 송금 횟수가 불필요하게 많아집니다.

**해결** — 각자의 순수 잔액을 계산한 뒤, 가장 많이 받아야 할 사람과 가장 많이 내야 할 사람을 차례로 상계하는 **그리디 방식**으로 송금 횟수를 줄였습니다. 정산 결과에서 바로 송금할 수 있도록 토스·카카오페이 딥링크를 연결했습니다.

### 3. SSE 연결이 끊기면 화면이 조용히 멈춤

**문제** — SSE는 네트워크가 잠깐 흔들려도 끊깁니다. 무작정 재연결하면 서버에 부담이 가고, 포기하면 사용자는 낡은 화면을 보면서도 그 사실을 모릅니다.

**해결** — 실패 원인을 나눠 처리했습니다.
([`frontend/src/hooks/useGroupStream.ts`](frontend/src/hooks/useGroupStream.ts))

- `401` · `403`은 재시도해도 복구되지 않으므로 즉시 폴백
- 그 외 오류는 점증 백오프(1초 → 최대 5초)로 3회 재시도
- 3회를 넘기면 5초 간격 폴링으로 전환해 데이터 갱신 자체는 유지
- 재연결에 성공하면 폴링 중단

폴백 시에는 React Query 캐시를 광역 무효화하지만, 평상시에는 **이벤트 타입별로 관련 캐시만 정밀 무효화**해 불필요한 재요청을 막았습니다.

---

## 🔭 아쉬웠던 점

백엔드는 테스트를 갖췄지만 **프론트엔드 테스트는 작성하지 못했습니다.**
SSE 폴백처럼 분기가 많은 로직일수록 테스트가 필요한데, 일정상 수동 확인에 그쳤습니다.

---

## ⚙️ 실행 방법

```bash
git clone https://github.com/hodu42/groutrip.git
cd groutrip

cp .env.example .env   # DB, OAuth 클라이언트 정보 설정

docker compose up -d
```
