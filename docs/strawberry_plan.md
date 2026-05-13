# 🫐 BlueberryLab


# 🍓 Strawberry — 서비스 개발 기획서

> 버전 0.2 | 작성일: 2026.04  
> 개인 서버(RAM 4GB / 120GB) 기반 자체 운영 플랫폼

---

## 1. 프로젝트 개요

### 비전

"내가 만들고, 내가 쓰고, 내가 키우는 개인 플랫폼"

### 목표

- 일상의 기록·학습·관리를 하나의 플랫폼에서 통합
- 기능별 독립 컨테이너로 운영하여 확장성과 안정성 확보
- Terraform + Docker + CI/CD 로 실제 프로덕션 수준의 개발 경험 습득

### 사용자

- 주 사용자: 본인 (peterhm)
- 확장 대상: 소수 지인 (초대 기반)

---

## 2. 시스템 아키텍처

```
[Mac / 외부 기기]
       │ HTTPS (Certbot → SSL)
       ▼
[DuckDNS] → strawberry.duckdns.org
       │
       ▼
[Nginx Reverse Proxy] ← 모든 요청의 진입점
       │
       ├── /habit, /lang, /links ...  → React 정적 파일 서빙
       │
       ├── /api/v1/auth/*             → auth-api (검증 제외)
       │
       └── /api/v1/*                  → auth/verify 검증 먼저
                │
                ├── 유효 → 각 앱으로 전달 (X-User-Id 헤더 포함)
                │     ├── /api/v1/habit/...   → habit-api
                │     ├── /api/v1/lang/...    → lang-api
                │     ├── /api/v1/monitor/... → monitor-api
                │     └── ...
                │
                ├── 무효 → 401 Unauthorized
   ┌────────────┴────────────────────────┐
   │         Docker Network              │
   │  ┌──────────┐ ┌──────────┐          │
   │  │ habit-api│ │ lang-api │  ...     │
   │  └──────────┘ └──────────┘          │
   │         ┌──────────┐                │
   │         │ 공용 DB   │                │
   │         │(Postgres)│                │
   │         └──────────┘                │
   └─────────────────────────────────────┘
```

### 핵심 원칙

- 각 앱 백엔드는 독립 컨테이너로 분리 → 한 앱이 죽어도 나머지에 영향 없음
- 프론트엔드(React)는 빌드 후 Nginx가 정적 파일로 서빙 (별도 컨테이너 불필요)
- 공용 PostgreSQL 하나에 앱별 스키마 분리
- Nginx가 경로 기반으로 각 컨테이너로 라우팅
---

## 3. 기술 스택

| 분류    | 기술                      | 선택 이유           |
| ----- | ----------------------- | --------------- |
| 컨테이너  | Docker + Docker Compose | 앱별 환경 격리        |
| 인프라   | Terraform               | 서버 설정을 코드로 관리   |
| CI/CD | GitHub Actions          | 코드 push 시 자동 배포 |
| 프록시   | Nginx                   | 단일 진입점, SSL 처리  |
| DB    | PostgreSQL              | 안정성, 앱별 스키마 분리  |
| 백엔드   | Python (FastAPI)        | 빠른 개발, 학습 친화적   |
| 프론트엔드 | React                   | FastApi와 검증된 조합  |
| 알림    | Telegram Bot            | 개발의 난이도 따라 선택  |
| SSL | Certbot                    | Https 자동 발급    |
| 모니터링 | Netdata | App 04 안정화 전까지 병행 운영, 이후 제거 예정 |
| 인증      | JWT (Access + Refresh Token) | 표준 인증 방식, 보안 관리 용이 |
| 도메인     | DuckDNS               | 무료 DDNS, 추후 유료 도메인 이전  |
| DB 마이그레이션 | Alembic            | 스키마 변경을 코드로 관리         |

---

## 4. URL 라우팅 설계
 
### 패턴
 
```
# 프론트엔드 — Nginx 정적 서빙
strawberry.duckdns.org/{앱명}
 
# 백엔드 — FastAPI
strawberry.duckdns.org/api/v1/{앱명}/...
```
 
### 전체 경로
 
| 앱        | 프론트엔드     | 백엔드                  |
| -------- | --------- | -------------------- |
| Habit    | /habit    | /api/v1/habit/...    |
| Lang     | /lang     | /api/v1/lang/...     |
| Links    | /links    | /api/v1/links/...    |
| Monitor  | /monitor  | /api/v1/monitor/...  |
| Papers   | /papers   | /api/v1/papers/...   |
| Files    | /files    | /api/v1/files/...    |
| Snippets | /snippets | /api/v1/snippets/... |
| Hub (AI) | /hub      | /api/v1/hub/...      |
| Auth | /api/v1/auth/... | (verify는 Nginx 내부 전용) |
 
---
 
## 5. DB 스키마 설계
 
### 공통 스키마 (common)

common 스키마 — Auth 컨테이너가 단독으로 관리
각 앱은 common 스키마에 직접 접근하지 않음
user_id는 Auth가 검증 후 헤더로 전달
 
```sql
-- 유저 테이블
common.users
├── id           UUID  PK
├── username     VARCHAR(50)   UNIQUE
├── email        VARCHAR(255)  UNIQUE
├── password     VARCHAR(255)  -- bcrypt 해시
├── is_active    BOOLEAN DEFAULT true
├── created_at   TIMESTAMP
└── updated_at   TIMESTAMP
 
-- 리프레시 토큰
common.refresh_tokens
├── id           UUID  PK
├── user_id      UUID  FK → users.id
├── token        VARCHAR(255)  UNIQUE
├── expires_at   TIMESTAMP
└── created_at   TIMESTAMP
```
 
**JWT 흐름**
```
로그인 (ID/PW)
  ↓
Access Token  (15~30분)  ← 매 요청마다 사용
Refresh Token (7~30일)   ← Access Token 만료 시 재발급
```
 
---
 
### App 01 — habit 스키마
 
```sql
-- 습관 정의
habit.habits
├── id           UUID  PK
├── user_id      UUID  FK → common.users.id
├── name         VARCHAR(100)
├── description  TEXT
├── color        VARCHAR(7)    -- 히트맵 색상 (#HEX)
├── goal         INTEGER DEFAULT 30  -- 월 목표 횟수 추가
├── is_active    BOOLEAN DEFAULT true
├── created_at   TIMESTAMP
└── updated_at   TIMESTAMP
 
-- 매일 체크인 기록
habit.checkins
├── id           UUID  PK
├── habit_id     UUID  FK → habits.id
├── user_id      UUID  FK → common.users.id
├── checked_date DATE
├── created_at   TIMESTAMP
└── UNIQUE (habit_id, checked_date)   -- 하루 한 번만
 
```
 
> Streak(연속 달성일)은 별도 컬럼 없이 checkins 데이터를 쿼리해서 계산
 
---
 
### App 04 — monitor 스키마
 
```sql
-- 서버 리소스 스냅샷 (5분마다 저장)
monitor.server_metrics
├── id           UUID  PK
├── cpu_percent  FLOAT
├── ram_percent  FLOAT
├── disk_percent FLOAT
└── recorded_at  TIMESTAMP
 
-- 컨테이너 상태 스냅샷
monitor.container_metrics
├── id             UUID  PK
├── container_name VARCHAR(100)
├── status         VARCHAR(50)   -- running / exited / paused
├── cpu_percent    FLOAT
├── ram_percent    FLOAT
└── recorded_at    TIMESTAMP
 
-- 알림 기록
monitor.alerts
├── id           UUID  PK
├── type         VARCHAR(50)  -- cpu / ram / disk / container / security
├── message      TEXT
├── is_resolved  BOOLEAN DEFAULT false
├── created_at   TIMESTAMP
└── resolved_at  TIMESTAMP

-- 보안 로그 추가
monitor.security_logs
├── id           UUID  PK
├── ip           VARCHAR(45)   -- IPv6 대응
├── type         VARCHAR(50)   -- login_fail / suspicious_request / not_found
├── path         VARCHAR(255)  -- 접근 URL
├── count        INTEGER       -- 누적 횟수
├── is_alerted   BOOLEAN DEFAULT false
├── created_at   TIMESTAMP
└── updated_at   TIMESTAMP
```
 
> 실시간 데이터는 FastAPI에서 `psutil` + `docker SDK`로 직접 조회 (DB 저장 없음)  
> DB에는 히스토리 데이터만 저장
 
---

## 6. 앱 목록 (v1.0)

### 🔴 App 00 — 인증 (Auth) (개발 우선순위 1)

**역할:** 모든 앱의 인증을 담당하는 중앙 인증 컨테이너.
Nginx가 모든 API 요청을 Auth에 먼저 검증 요청하고,
각 앱은 JWT를 직접 처리하지 않음

**핵심 기능:**
- 로그인 (ID/PW → Access Token + Refresh Token 발급)
- Access Token 재발급 (Refresh Token 사용)
- 로그아웃 (Refresh Token 삭제)
- 토큰 검증 (Nginx 내부 전용 /auth/verify)
- 검증 후 user_id를 헤더에 담아 각 앱으로 전달

**API:**
- POST /api/v1/auth/login
- POST /api/v1/auth/refresh
- POST /api/v1/auth/logout
- GET  /auth/verify  ← Nginx 내부 전용, 외부 접근 불가

### 🔴 App 01 — Habit Tracker

**역할:** 매일 반복할 습관을 기록하고 시각화하는 앱. 달성률과 패턴을 한눈에 파악


**핵심 기능:**

- 습관 관리
  - 습관 생성 / 수정 / 삭제
  - 습관별 월 목표 횟수 (Goal) 설정
  - 습관별 아이콘 / 색상 설정
  - 매일 체크인
  - 연속 달성일 (Streak) 계산
- 대시보드
  - Daily Completion Rate 라인 차트
  - Monthly Progress 도넛 차트
  - 주차별 Overview (달성 수 + 달성률)
  - Top Habits 랭킹
  - Overall Progress (Done / Left / 달성률 / 바 차트)
- 히트맵 테이블
  - 습관 × 날짜 그리드
  - 날짜별 체크 표시
  - 연도 / 월 필터

**유저별 데이터 분리:** 모든 데이터는 `user_id`로 필터링하여 각자의 데이터만 접근

**확장 가능성:**

- 텔레그램 봇으로 매일 아침 리마인더
- HUB 연동 → 습관 패턴 분석

---

### 🟢 App 02 — 언어 섀도잉 (Language Shadowing)

**목적:** 영어 / 일본어 문장을 매일 학습

**핵심 기능:**

- 문장 저장 (한국어 / 영어 / 일본어)
- 오늘의 문장 랜덤 제공
- 학습 기록 (언제, 어떤 문장)
- 즐겨찾기 / 복습 리스트

**확장 가능성:**

- TTS(Text-to-Speech) 연동 → 발음 듣기
- AI 허브 → 문장 생성, 난이도 조절
- 텔레그램으로 매일 오늘의 문장 발송

---

### 🟢 App 03 — 링크 저장소 (Link Vault)

**목적:** 나중에 읽을 링크, 참고할 URL 모아두기

**핵심 기능:**

- URL 저장 + 자동 미리보기 (제목, 썸네일)
- 태그 분류
- 검색
- 읽음 / 안 읽음 상태 관리

**확장 가능성:**

- 파일 저장소 연동 → 스크린샷 자동 저장
- AI 허브 → 링크 자동 분류, 요약

---

### 🟢 App 04 — 서버 모니터링 (Monitor)

**역할:** Strawberry 서버 및 전체 컨테이너 상태를 실시간으로 감시하는 인프라 필수 앱. 다른 앱들이 살아있는지 가장 먼저 확인할 수 있어야 하므로 최우선 개발
 
**핵심 기능:**
 
- 실시간 대시보드
  - CPU / RAM / 디스크 현재 사용량
  - 각 컨테이너 상태 (running / exited / paused)
  - 최근 24시간 리소스 그래프
- 알림 임계치 관리
  - 대시보드에서 임계치 직접 조정 가능
  - CPU(5분 평균) 경고 80% / 위험 90%
  - RAM 경고 85% / 위험 95%
  - 디스크 경고 80% / 위험 90%
  - 컨테이너 다운 즉시 알림
- 히스토리
  - 5분마다 스냅샷 저장
  - 7일치 보관 후 자동 삭제
- 알림
  - 경고 / 위험 단계별 텔레그램 알림
  - 알림 이력 기록 및 해결 여부 관리
- 보안 로그
  - 비정상 요청 패턴 감지 (단시간 대량 요청)
  - 존재하지 않는 URL 반복 접근 감지
  - 의심 IP 텔레그램 알림
  - 로그인 실패 횟수 기록
  - 특정 IP 5회 이상 실패 시 텔레그램 알림
**기술 구현:**
- 실시간 데이터: `psutil` (서버 리소스) + `docker SDK` (컨테이너 상태)
- 히스토리: PostgreSQL `monitor` 스키마
**확장 가능성:**
- AWS 등 외부 서버도 추가 모니터링
- 로그 수집 및 분석


---

### 🟢 App 05 — 논문 기록 (Paper Archive)

**목적:** 읽은 논문 / 읽을 논문 관리

**핵심 기능:**

- 논문 메타데이터 저장 (제목, 저자, 연도, 저널)
- PDF 파일 첨부 (파일 저장소 연동)
- 상태 관리 (읽는 중 / 완료 / 읽고 싶음)
- 메모 / 하이라이트 기록

**확장 가능성:**

- AI 허브 → PDF 자동 요약
- 인용 형식 자동 생성

---

### 🟢 App 06 — 파일 저장소 (File Storage)

**목적:** 개인 클라우드 (Google Drive 대체)

**핵심 기능:**

- 파일 업로드 / 다운로드
- 폴더 구조 관리
- 다른 앱들과 공유 가능한 공용 저장소 역할
- 사용량 대시보드

**용량 계획:**

- OS + Docker: ~15GB
- 앱 데이터: ~5GB
- 파일 저장소: ~100GB 활용 가능

**확장 가능성:**

- 외부 접근 시 링크 공유 기능
- 자동 백업 (AWS S3 등)

---

### 🟢 App 07 — 코드 스니펫 (Code Snippet)

**목적:** 자주 쓰는 코드 조각 저장

**핵심 기능:**

- 코드 저장 + 언어별 신택스 하이라이팅
- 태그 / 언어별 분류
- 검색
- 복사 버튼

**확장 가능성:**

- AI 허브 → 코드 설명 자동 생성
- GitHub Gist 연동

---

### 🟡 App 08 — AI 허브 (AI Hub)

**목적:** 모든 앱을 AI로 연결하는 중앙 두뇌

**핵심 기능:**

- 채팅 인터페이스
- 각 앱 데이터에 접근하여 질문 응답
  - "이번 주 습관 어때?" → Habit DB 조회
  - "논문 요약해줘" → Paper Archive + File Storage
  - "오늘 공부할 문장 추천해줘" → Shadowing 앱 연동
- 텔레그램 봇 연동

**AI 엔진 옵션:**

- Google AI Studio API (무료, 하루 1,500 요청)
- Anthropic Claude API (유료, 고품질)
- 추후 오픈소스 모델 자체 호스팅 검토

**확장 가능성:**

- 앱이 늘어날수록 AI 허브의 기능도 자동 확장
- 음성 입력 연동

---

## 7. 개발 로드맵

### Phase 1 — 기반 구축 (인프라)

- [ ] Terraform으로 서버 초기 설정 코드화
- [ ] Docker Compose 기본 구성
- [ ] Nginx Reverse Proxy 설정
- [ ] PostgreSQL 컨테이너 구성
- [ ] GitHub Actions CI/CD 파이프라인 구성
- [ ] DuckDNS 도메인 등록 + Certbot SSL 설정

### Phase 2 — 첫 번째 앱 (MVP)

- [ ] App 00 Auth (모든 앱의 전제 조건)
- [ ] App 04 모니터링 (가장 단순, 인프라 검증용)
- [ ] App 01 Habit Tracker

### Phase 3 — 핵심 앱 완성

- [ ] App 06 파일 저장소
- [ ] App 03 링크 저장소
- [ ] App 05 논문 기록
- [ ] App 07 코드 스니펫

### Phase 4 — 고급 기능

- [ ] App 02 언어 섀도잉
- [ ] App 08 AI 허브
- [ ] 텔레그램 봇 전면 연동

---

## 8. 디렉토리 구조 (예상)

```

strawberry/                     # 루트
├── docs/
│   ├── strawberry_plan.md      # 개발자 기획서
│   └── strawberry_business.md  # 경영학적 기획서 (예정)
├── terraform/                  # 인프라 코드
├── nginx/
│   └── conf.d/
├── docker-compose.yml
├── apps/
│   ├── auth/           
│   │    └── backend/
│   ├── habit/
│   │   ├── backend/
│   │   └── frontend/
│   ├── lang/
│   ├── links/
│   ├── monitor/
│   ├── papers/
│   ├── files/
│   ├── snippets/
│   └── hub/
└── .github/
    └── workflows/

```

---

## 9. 확장 가능성 고려사항

- **새 앱 추가:** `apps/` 아래 폴더 추가 + `docker-compose.yml` 에 서비스 한 줄 추가
- **사용자 추가:** 인증 시스템 (App 공통 Auth 컨테이너) 으로 초대 기반 관리
- **외부 공개:** Nginx에 도메인 + SSL 추가만으로 가능
- **클라우드 이전:** Docker 기반이라 AWS / GCP 이전 용이
- **앱 교체:** 컨테이너 단위로 교체 가능, 다른 앱에 영향 없음
- **스키마 변경:** Alembic 마이그레이션으로 안전하게 관리

---

> 이 문서는 개발자 관점의 살아있는 문서입니다. 개발하면서 계속 업데이트 예정.  
> 경영학적 관점은 `strawberry_business.md` 참고.
