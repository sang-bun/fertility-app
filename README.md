# 🧬 Fertility AI Server - AI/LLM 오케스트레이션 백엔드 포트폴리오

> **💡 본 레포지토리는 팀 프로젝트 [fertility-app](https://github.com/Capstone-Fertility-AI/fertility-app)를 기반으로, 백엔드 개발자 [영준]의 기여도를 중심으로 재구성된 포트폴리오용 레포지토리입니다.**

## 📌 1. 프로젝트 개요 및 나의 역할

- **프로젝트 한줄 소개:** 사용자의 신체·생활습관 데이터를 바탕으로 AI가 난임 위험을 분석하고, LLM이 맞춤형 웰니스 리포트를 제공하는 헬스케어 백엔드 시스템
- **개발 기간:** 2026.03 ~ 현재진행중
- **나의 역할:** Back-end (인증/인가, AI/LLM 연동 오케스트레이션, 세션 상태 관리 및 클라우드 배포)
- **기여도 요약:** 카카오 OAuth 및 자체 로그인 인프라를 구축하고, 사용자의 설문부터 AI 위험도 예측(FastAPI), 그리고 LLM 맞춤 리포트 생성(GPT-4o-mini)에 이르는 복잡한 워크플로우를 Spring WebFlux(`WebClient`)를 활용해 안정적으로 오케스트레이션했습니다. 또한 AWS EC2와 RDS를 활용한 인프라 구축 및 배포를 담당했습니다.

## 🛠 2. 활용 기술 스택 (Tech Stack)

- **Language & Framework**
  - `Java 17`, `Spring Boot 3.5.x`
- **Database & ORM**
  - `PostgreSQL` (AWS RDS)
  - `Spring Data JPA` + `Hibernate`
- **Security & Authentication**
  - `Spring Security`
  - `JWT` (Access/Refresh)
  - `Spring OAuth2 Client` (카카오 연동)
- **API & External Integration**
  - `Spring WebFlux (WebClient)` (FastAPI 및 OpenAI `gpt-4o-mini` 비동기 호출)
  - `SpringDoc OpenAPI` (Swagger)
  - `Spring Cloud AWS` (S3)
- **Infra & Ops**
  - `AWS EC2 (Ubuntu)`
  - `Spring Boot Actuator`

## 🔥 3. 주요 담당 업무 및 기여도 (My Contributions)

### [인증/보안] 이중 로그인 지원 및 JWT Stateless 인프라 구축
- **Spring Security + JWT 기반 인증 체계:** `JwtTokenProvider`, `JwtAuthenticationFilter`를 설계하여 `/oauth/**`, `/auth/signup` 등은 `permitAll`로, 그 외 보호 API는 `Authorization: Bearer` 토큰 방식으로만 접근 가능한 Stateless 환경 구축.
- **자체 및 카카오 OAuth 로그인 구현:** `UserCommandServiceImpl`에서 BCrypt 기반 자체 회원가입 구현. 카카오 로그인 시 인가 URL 생성부터 토큰 발급(`OauthUserService.handleKakaoUser`)까지의 전 과정을 구현.
- **프론트엔드 연동 최적화:** 카카오 콜백 처리 후 JSON 응답 대신 `302 Redirect`를 통해 프론트엔드의 특정 URL(`frontend-callback-url`)로 이동시키며, 쿼리 파라미터(`accessToken`, `refreshToken`, `userId`, `nickname`)로 토큰을 안전하게 전달하도록 구현.
- **토큰 갱신 플로우 지원:** `POST /auth/refresh` API를 통한 Access Token 만료 갱신 프로세스 구축.

### [검사 세션] 멀티스텝 설문의 시작점과 상태 모델 관리
- **검사 세션(Test Session) 파이프라인 설계:** `POST /tests/start`에서 성별 입력 시 고유 `TestSession`을 생성하여 `sessionId`를 반환. 이후 클라이언트의 단계별 PATCH 요청 데이터를 하나의 세션 레코드에 일관되게 누적(`currentStep`) 관리.
- **사용자별 정보 보안 격리:** `GET /users/me` 호출 시 `CustomPrincipal`에서 추출한 `userId` 기반으로만 정보를 조회하도록 설계하여 타 사용자 데이터 접근 원천 차단.

### [AI 오케스트레이션] 외부 AI 추론 서버(FastAPI) 연동 및 비즈니스 데이터 가공
- **데이터 정규화 및 파이프라인 변환:** 설문 최종 제출(`POST /tests/{sessionId}/submit`) 시, PSS(스트레스 10문항) 합계를 계산하고 구간화(`LOW`/`NORMAL`/`HIGH`)하여 세션 상태를 `COMPLETED`로 전환. 프론트의 한글 범주형 데이터(흡연/음주 등)를 AI 서버가 기대하는 정수 스케일(0~2)로 변환하는 `AiLifestyleCategoryMapper` 구현.
- **WebClient 비동기 통신 및 장애 대비:** 전용 빈(`aiWebClient`)을 생성하여 FastAPI 서버 호출 시 타임아웃을 명시적으로 제어. AI 기능 오프 스위치(`ai.prediction.enabled=false`)를 마련하여 AI 서버 통신 장애 시 개발/테스트용 더미 데이터를 반환하는 Fallback 로직 적용.
- **결과 매핑 및 영속화:** AI 응답으로 받은 가변 길이의 `top_factors`를 `StringListJsonConverter`로 직렬화하여 PostgreSQL에 안전하게 저장. AI의 원시 점수(`score`)를 Spring 백엔드 자체 정책 기반의 위험 등급(`RiskLevel.determineLevel`: SAFE/WARNING/DANGER)으로 매핑.

### [LLM 오케스트레이션] 맞춤형 상세 웰니스 리포트 생성 분리
- **관심사 분리 및 비동기 처리:** AI 예측 결과 반환과 LLM 리포트 생성을 별도의 엔드포인트(`GET /api/results/{resultId}`)로 분리하여 사용자 체감 응답 시간을 단축.
- **OpenAI 프롬프트 엔지니어링 구현:** `ReportSystemPrompt`를 정의하여 시스템 페르소나 및 마크다운 템플릿을 System 메시지로 고정. 이후 `nickname`, `score`, `stressLevel`, `factors` 등 사용자 맞춤 데이터를 JSON 형태의 User 메시지로 주입하여 `OpenAI Chat Completions API (gpt-4o-mini 모델)`를 호출해 리포트 생성을 유도.

### [인프라] 운영 환경 배포 및 환경 설정 보안
- **클라우드 인프라 구축:** AWS EC2(Ubuntu) 기반 서버 환경 구성 및 AWS RDS PostgreSQL 연동으로 데이터 영속성 확보.
- **보안 격리:** GitHub Secrets 환경 변수 주입 방식을 채택하여 DB, JWT, OAuth Secret 정보 등 민감 설정 분리 및 코드 노출 방지 (`llm` 속성을 `ai`와 동일 레벨로 격리하여 바인딩 이슈 해결).

## 💡 4. 트러블 슈팅 (Troubleshooting)

### 🎯 [연동/운영] AI 응답 `top_factors` 고정 길이 의심 현상 추적 및 디버깅
- **🚨 문제 상황:** 통합 테스트 시 가변 길이가 정상인 `top_factors` 리스트가 운영 환경에서 지속적으로 크기 3으로만 반환되는 문제가 발생하여, 프론트/백엔드/AI 간 책임 소재 파악에 혼선이 있었습니다.
- **💡 해결 방법:** Spring 디버그 로그와 로컬 cURL 테스트를 통해 백엔드(Spring) 로직에는 문제가 없음을 우선 증명했습니다. 이후 EC2 서버 점검을 통해 구버전 Uvicorn 프로세스 잔존 혹은 가상 환경(venv) 패키지 충돌 문제를 파악하고, `python -m uvicorn` 명령으로 올바른 런타임 환경에서 AI 서버를 재기동했습니다.
- **📈 결과:** 스펙과 일치하는 0~12개 가변 길이 응답 정상화를 확인했으며, 이를 계기로 추후 런타임 환경 증명을 위한 배포 커밋 해시 로깅 시스템을 도입했습니다.

### 🎯 [아키텍처/UX] 설문 제출 API 지연 및 외부 장애 전파(Cascading Failure) 완화
- **🚨 문제 상황:** 사용자가 검사를 제출할 때 1) DB 저장, 2) AI 예측 API 호출, 3) OpenAI LLM 호출 로직이 순차적으로 결합되어 있어, 응답 대기 시간이 매우 길어졌으며 LLM 서버 장애 시 설문 완료 자체가 실패하는 구조적 한계가 있었습니다.
- **💡 해결 방법:** 핵심 도메인 로직과 부가적인 리포트 생성을 분리했습니다. 설문 제출 시점에는 동기로 빠른 AI 예측 점수와 등급 판정만 저장 및 반환하고, 무거운 처리인 LLM 호출은 사용자가 리포트 화면을 조회할 때(`GET /api/results/{resultId}`) 비동기로 수행하도록 아키텍처를 재설계했습니다.
- **📈 결과:** "점수/위험 요인은 즉시, 긴 리포트는 별도 조회"라는 효율적인 UX 흐름을 구축하였으며, 향후 발생할 수 있는 OpenAI API 타임아웃 장애로부터 핵심 메인 비즈니스(검사 제출 파이프라인)를 안전하게 격리시켰습니다.

## 🔗 5. 링크

- **팀 프로젝트 원본 레포지토리:** [fertility-app](https://github.com/Capstone-Fertility-AI/fertility-app)
