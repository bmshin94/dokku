# 📘 Dokku 분석 정리 (한국어)

> 이 문서는 Dokku 저장소를 직접 분석하고 정리한 한국어 요약본입니다.
> 작성일: 2026-09-17

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| 원본 저장소 (upstream) | https://github.com/dokku/dokku |
| 내 포크 (this repo) | https://github.com/bmshin94/dokku |
| 공식 문서 | https://dokku.com/docs/getting-started/installation/ |
| 커뮤니티 (Slack) | https://slack.dokku.com/ |
| 후원 | https://opencollective.com/dokku |

---

## 1. Dokku란?

> **"Docker powered mini-Heroku. The smallest PaaS implementation you've ever seen."** — README.md

한마디로 **내 서버에 직접 설치하는 미니 Heroku**입니다.
`git push` 한 번으로 웹 앱이 빌드되고, 컨테이너로 실행되고, 도메인까지 연결됩니다.

| 항목 | 내용 |
|---|---|
| 라이선스 | MIT (Copyright © 2014 Jeff Lindsay) |
| 분석 시점 버전 | v0.38.27 |
| 구현 언어 | Bash + Go (Go 파일 319개, `go.work` 워크스페이스로 36개 모듈 관리) |
| 문서 | `docs/` 아래 마크다운 110개 |
| 지원 OS | Ubuntu 22.04 / 24.04, Debian 11·12·13 (amd64 / arm64) |

---

## 2. 폴더 구조

| 경로 | 역할 |
|---|---|
| `plugins/` | **핵심.** 48개 플러그인. Dokku의 모든 기능이 여기 부품으로 분리돼 있음 |
| `docs/` | 공식 문서 원본 (dokku.com 사이트 소스) |
| `dokku` | CLI 본체 스크립트 (`dokku apps:create` 등을 처리) |
| `bootstrap.sh` | 서버 설치 스크립트 |
| `contrib/dokku_client.sh` | 로컬에서 원격 Dokku를 조작하는 공식 클라이언트 |
| `tests/` | bats 기반 배포 시나리오 테스트 + shellcheck |
| `Vagrantfile` | 로컬 실습용 가상 서버 설정 |
| `Dockerfile`, `debian/`, `*.mk` | 패키징 (deb 패키지 / 도커 이미지 빌드) |
| `.github/`, `.circleci/` | CI 파이프라인 |
| `docs/enterprise/pro.md` | 상용 제품 **Dokku Pro** 안내 |

### 플러그인 설계가 특히 훌륭한 부분

```
plugins/
├── builder-dockerfile / builder-herokuish / builder-pack
│   └── builder-nixpacks / builder-railpack / builder-lambda   → "어떻게 빌드할까"
├── scheduler-docker-local / scheduler-k3s / scheduler-null    → "어디서 실행할까"
├── nginx-vhosts / caddy-vhosts / traefik-vhosts / haproxy-vhosts → "어떤 프록시를 쓸까"
└── certs, domains, config, storage, cron, ps, logs, checks, resource ...
```

빌더 / 스케줄러 / 프록시를 **통째로 교체 가능한 구조**입니다.

---

## 3. 동작 흐름

```
내 노트북                          내 서버 (Dokku 설치됨)
─────────                         ─────────────────────
git push dokku main  ──────────▶  ① git 플러그인이 push 감지
                                  ② builder가 언어/Dockerfile 자동 감지
                                  ③ Docker 이미지 빌드
                                  ④ scheduler가 컨테이너 실행
                                  ⑤ nginx 등 프록시가 도메인 연결 + SSL
                                  ⑥ 헬스체크 통과 후 무중단 전환
                                  ──▶ https://myapp.내도메인.com ✅
```

---

## 4. 설치 및 사용법

### 요구사항 (`bootstrap.sh`에서 확인)
- Ubuntu 22.04 / 24.04 또는 Debian 11 / 12 / 13
- 메모리 **1GB 이상 권장** (미만이면 스크립트가 swap 설정을 경고)
- hostname이 설정돼 있어야 함
- SSH 키 한 쌍

### STEP 1 — 설치 (서버에서)
```bash
wget -NP . https://dokku.com/install/v0.38.27/bootstrap.sh
sudo DOKKU_TAG=v0.38.27 bash bootstrap.sh
```

### STEP 2 — 초기 설정
```bash
dokku ssh-keys:add admin ~/.ssh/id_rsa.pub
dokku domains:set-global mydomain.com
```

### STEP 3 — 앱 배포
```bash
# 서버에서
dokku apps:create myapp
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
dokku postgres:create mydb
dokku postgres:link mydb myapp        # DATABASE_URL 자동 주입
dokku config:set myapp SECRET_KEY=abcd1234

# 내 컴퓨터에서
git remote add dokku dokku@mydomain.com:myapp
git push dokku main                   # 배포 완료
```

### STEP 4 — 운영 명령어
```bash
dokku logs myapp -t          # 실시간 로그
dokku ps:restart myapp       # 재시작
dokku ps:scale myapp web=3   # 스케일 아웃
dokku domains:add myapp www.mydomain.com
dokku enter myapp web        # 컨테이너 진입
dokku apps:destroy myapp     # 삭제
```

### 원격 실행 (서버 접속 없이)
```bash
ssh -t dokku@mydomain.com logs myapp -t
```
`contrib/dokku_client.sh`를 설치하면 로컬에서 `dokku logs myapp` 형태로 바로 사용 가능합니다.

---

## 5. 플러그인? 스킬? MCP? → **전부 아님**

| 구분 | 정체 | 실행 위치 | 사용 주체 |
|---|---|---|---|
| Claude 플러그인 / 스킬 | AI에게 능력을 추가 | 로컬 Claude Code | AI |
| MCP | AI ↔ 외부 서비스 연결 규격 | AI와 외부 프로그램 사이 | AI |
| **Dokku** | **서버 운영 소프트웨어** | **리눅스 서버** | **사람(개발자)** |

Dokku는 AI와 무관한 전통적인 인프라 도구입니다.
`plugins/` 폴더가 있어 혼동하기 쉽지만, 그것은 **Dokku 자신의 내부 부품**을 뜻합니다.

> 참고: Dokku를 AI와 연결하려면 별도로 MCP 서버를 직접 만들어야 하며, 이 저장소에는 그런 기능이 없습니다.

---

## 6. API 토큰이 필요한가? → **필요 없음**

Dokku의 인증은 **SSH 공개키 기반**입니다.

```bash
dokku ssh-keys:add admin ~/.ssh/id_rsa.pub
```

토큰 발급·만료·갱신 개념이 없습니다. 다만 아래는 예외적으로 자격증명이 필요합니다.

| 상황 | 필요한 것 |
|---|---|
| 외부 도커 레지스트리 사용 | 레지스트리 계정 정보 (`docs/advanced-usage/registry-management.md`) |
| GitHub Actions 자동 배포 | SSH 개인키를 GitHub Secret으로 저장 |
| Let's Encrypt SSL | 이메일 주소 (토큰 아님) |
| 와일드카드 SSL | DNS 업체 API 토큰 |

**중요:** Dokku 오픈소스판에는 **HTTP REST API가 내장돼 있지 않습니다.** 모든 조작은 SSH 명령으로 이뤄집니다.
(단, 아래 Dokku Pro는 JSON API를 제공합니다.)

---

## 7. 왜 GitHub에서 유명한가

1. **타이밍** — 2022년 Heroku 무료 플랜 종료 후 대안 수요 폭발
2. **전설적인 단순함** — 2013년 Bash 100줄 남짓으로 시작한 "가장 작은 PaaS"
3. **쿠버네티스 피로감** — `docker run` 수동 관리와 k8s 사이의 "딱 좋은 중간 지점"
4. **문서 품질** — 마크다운 110개 분량의 체계적인 공식 문서
5. **플러그인 생태계** — PostgreSQL, Redis, MongoDB, ClickHouse 등 공식·커뮤니티 플러그인 다수

> 별(Star) 수치는 이 분석 시점에 직접 조회하지 않았습니다. 정확한 수치는 원본 저장소에서 확인하세요.

---

## 8. AI 에이전트 운영에 도움이 되는가

- **에이전트의 "두뇌"를 만드는 것**과는 무관 ❌
- **만든 에이전트를 24시간 배포·운영하는 인프라**로는 매우 적합 ✅

| 에이전트 운영 요구사항 | 대응 Dokku 기능 |
|---|---|
| API 키 안전 보관 | `dokku config:set myagent ANTHROPIC_API_KEY=...` |
| 주기적 작업 실행 | `cron` 플러그인 |
| 대화 기록 / 상태 저장 | `postgres`, `redis` 플러그인 |
| 벡터 DB 데이터 영속화 | `storage` 플러그인 |
| 로그 모니터링 | `dokku logs myagent -t` |
| 죽으면 자동 복구 | `checks` 플러그인 (헬스체크) |

**주의:** LLM 모델을 서버에서 직접 구동(GPU 필요)하는 용도로는 부적합합니다.
API 호출형 에이전트에는 매우 잘 맞습니다.

---

## 9. React / PHP와의 관계

### A. React·PHP 앱을 Dokku로 배포하기 → **완전 가능**

React 예시 (Dockerfile이 있으면 Dokku가 자동 감지):
```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

PHP는 `composer.json` 기반 Heroku 빌드팩 또는 `php-fpm + nginx` Dockerfile로 배포합니다.
Laravel, WordPress 모두 가능합니다.

> ⚠️ 앱이 `PORT` 환경변수를 존중해야 합니다. (공식 문서의 경고 사항)

### B. Dokku 자체를 React/PHP로 재구현하기 → **비권장**

- PHP: 기술적으로 가능하나 docker/nginx/systemd 같은 시스템 명령을 계속 호출해야 해 비효율적
- React: 브라우저 UI 도구라 서버 제어 불가

### C. 권장 방향 — Dokku 위에 UI 얹기

```
React 대시보드  ──(REST)──▶  Node/PHP 백엔드  ──(ssh)──▶  Dokku
```
`docs/deployment/remote-commands.md`에 따르면 모든 Dokku 명령은 SSH로 원격 실행 가능하므로
이 구조가 성립합니다. 단, 아래 Dokku Pro와 경쟁 관계임을 유의하세요.

---

## 10. ⚠️ 중요 — Dokku Pro (상용 제품)가 이미 존재

`docs/enterprise/pro.md` 확인 결과, 원작자 팀이 **유료 상용 제품**을 판매 중입니다.

Dokku Pro가 제공하는 기능:
- SPA 웹 UI (앱 / 데이터스토어 / SSH 키 관리)
- **JWT 인증 기반 JSON API**
- HTTPS 기반 git push (SSH가 차단된 환경용)
- 단일 바이너리 배포, 월 단위 릴리즈
- Debian / RPM 패키지로 제공, 라이선스 검증을 위해 인터넷 연결 필요

**시사점**
- ❌ "웹 UI가 없으니 만들어 팔자"는 전제는 틀렸음 — 정면 경쟁 대상이 원작자 팀
- ✅ 반대로 **유료 수요가 검증됐다**는 증거
- ✅ 한국어 지원 없음, 국내 결제 수단 없음, 오프라인 환경 제약 → 차별화 여지 존재

---

## 11. 라이선스 (수익화 관점)

`LICENSE` 원문에 **"sell"** 권한이 명시된 MIT 라이선스입니다.

| 항목 | 가능 여부 |
|---|---|
| 상업적 판매 | ✅ 명시적 허용 |
| 수정 후 자사 제품화 | ✅ |
| 호스팅 서비스로 재판매 | ✅ |
| 소스 공개 의무 | ❌ 없음 (GPL이 아님) |

**의무 및 주의사항**
1. 저작권 고지문(`Copyright (C) 2014 Jeff Lindsay`)을 복사본에 포함해야 함
2. **"AS IS" 무보증 조항** → 서비스로 판매 시 책임은 전적으로 판매자에게 있음. 계약서에 면책·SLA 범위 명시 필요
3. "Dokku" 명칭을 자사 제품명으로 사용하는 것은 지양. "Dokku 기반"이라는 표기는 무방

---

## 12. 수익화 아이디어

| # | 아이디어 | 난이도 | 첫 수익까지 | 확장성 |
|---|---|---|---|---|
| 1 | 배포 대행 · 구축 컨설팅 | ⭐ | 2~4주 | 낮음 |
| 2 | 한국어 교육 콘텐츠 | ⭐⭐ | 1~3개월 | 높음 |
| 3 | 업종 특화 턴키 서비스 | ⭐⭐⭐ | 2~3개월 | 중간 |
| 4 | 관리형 호스팅 SaaS | ⭐⭐⭐⭐⭐ | 6개월+ | 매우 높음 |
| 5 | 니치 플러그인 판매 | ⭐⭐⭐ | 3~6개월 | 중간 |
| 6 | 배포형 보일러플레이트 | ⭐⭐⭐ | 1~2개월 | 높음 |
| 7 | 내 인프라 비용 절감 | ⭐ | 즉시 | — |

### 1) 배포 대행 · 구축 컨설팅 — 가장 빠른 현금화
- **타겟**: 인프라를 모르는 1인 개발자, 외주 후 서버 관리가 막막한 소상공인, 초기 스타트업
- **제공**: 서버 세팅 → Dokku 설치 → 도메인·SSL → DB·백업 → CI/CD 연동 → 사용 교육
- **가격 예시**: 기본 구축 30~50만원 / 앱 추가 5~10만원 / 월 유지보수 5~15만원
- **마진 포인트**: 실작업 2~4시간이면 끝나지만 고객에겐 며칠 걸릴 일
- **리스크**: 장애 대응 부담 → SLA(대응 시간대)를 계약서에 반드시 명시

### 2) 한국어 교육 콘텐츠 — 확장성 최고
- 공식 문서 110개가 전부 영어이고 한국어 자료가 희소함
- 무료(블로그·유튜브)로 유입 → 전자책(1.5~3만원) / 강의(5~8만원) / 1:1 코칭으로 수익화
- 커리큘럼 초안: 왜 Dokku인가 → VPS 선택 → 설치 → 첫 배포 → DB 연결 → 도메인·SSL →
  무중단 배포 → CI/CD → 백업 전략 → 실전(AI 에이전트 운영)
- **선순환**: 강의 수강생이 곧 1)의 구축 대행 고객이 됨

### 3) 업종 특화 턴키 서비스
Dokku를 전면에 내세우지 않고 완제품으로 판매합니다.
- 소규모 쇼핑몰 호스팅 (월 3~5만원 구독)
- 학원·부트캠프 실습 환경 (서버 1대로 수강생 30명 분 앱 운영, 기관 연 단위 계약)
- **AI 에이전트 호스팅** (cron + storage + config 조합, 월 2~5만원 구독)

### 4) 관리형 호스팅 SaaS — 잠재력 최대, 난이도 최상
- 수익 구조 예시: VPS 1대(월 4만원)에 15~25명 수용 × 9,900원 → 서버당 월 순익 11~21만원
- **장벽**: 결제 시스템, 멀티테넌시 보안 격리, 24시간 장애 대응, Dokku Pro와의 경쟁
- 1~3번으로 노하우와 고객을 쌓은 뒤 도전할 것

### 5) 니치 플러그인 판매
`docs/development/plugin-creation.md`에 따르면 **플러그인 구현 언어에 제한이 없습니다.**
`plugin.toml` 작성 + 실행 권한 부여만 하면 됩니다.
- 한국형 백업 플러그인 (네이버 클라우드 / 카카오 오브젝트 스토리지)
- 카카오톡·슬랙 배포 알림 플러그인
- 모니터링 대시보드, 자원 사용량 리포트
- 모델: 오픈소스 무료 배포로 인지도 확보 → Pro 버전 유료화
- **현실**: 개발자 대상 유료 도구는 국내 시장이 작아 글로벌 타겟이 유리

### 6) 배포형 보일러플레이트
- React/Next.js + 백엔드 + 인증 + 결제 + `Dockerfile` + `deploy.sh` 패키지
- 차별점: 경쟁 제품은 "Vercel에 올리세요"(월 고정비) → 본 제품은 "5천원 VPS에 올리세요"
- 가격: $49~199 (Gumroad 등) 또는 국내 5~15만원

### 7) 내 인프라 비용 절감 (간접 수익)
| 기존 | 전환 후 |
|---|---|
| Vercel Pro + Supabase Pro + Redis 호스팅 ≈ 월 7만원 | VPS 1대 ≈ 월 1만원 |

연 약 70만원 절감. 사이드 프로젝트를 늘려도 추가 비용이 발생하지 않는 것이 핵심 이점.

### 90일 실행 플랜 (권장)
```
1~2주차    내 서버에 Dokku 구축 + 개인 프로젝트 2~3개 배포 (7번 즉시 실현, 포트폴리오 확보)
3~6주차    블로그 시리즈 5편 + 유튜브 1편 (2번 씨앗 뿌리기)
7~10주차   크몽·숨고에 구축 대행 서비스 오픈 (1번 첫 현금화)
11~13주차  고객 피드백 반영해 전자책·강의 초안 완성 (2번 본격 런칭)
이후       반응 좋은 방향으로 3·6번 확장 → 여력이 되면 4번 도전
```

### 리스크 체크리스트
| 리스크 | 대응 |
|---|---|
| MIT 무보증 조항 | 고객 계약서에 면책·SLA 범위 명시 |
| 단일 서버 = 단일 장애점 | 백업 자동화 + 복구 절차 문서화 (`docs/advanced-usage/backup-recovery.md`) |
| "Dokku" 명칭 사용 | 제품명으로 사용 금지, "Dokku 기반" 표기는 가능 |
| Dokku Pro와 경쟁 | 정면승부 회피 → 한국어·업종 특화·서비스로 차별화 |
| 새벽 장애 대응 | 대응 시간대를 계약에 명시 |
| 국내 시장 규모 | 개발자 대상 제품보다 서비스·교육이 안전 |

> 시장 규모와 실제 수요는 별도 검증이 필요합니다. 크몽·인프런에서 "배포", "서버 구축" 키워드의
> 경쟁 상품 수와 리뷰 수를 직접 확인해 보는 것을 권장합니다.

---

## 13. 최종 요약

| 질문 | 답 |
|---|---|
| Dokku란? | Docker 기반 셀프호스팅 PaaS, "가장 작은 Heroku" |
| 설치·사용 | 리눅스 서버에 스크립트 2줄로 설치 → `git push`로 배포 |
| 플러그인·스킬·MCP인가? | **아님.** AI와 무관한 서버 운영 도구 |
| API 토큰 필요? | **불필요.** SSH 키 기반 인증 (Pro 버전은 JSON API 제공) |
| 왜 유명한가? | Heroku 유료화 + 압도적 단순함 + 뛰어난 문서 |
| AI 에이전트에 도움? | 에이전트 **배포·운영 인프라**로 매우 적합 |
| React/PHP | 배포는 완전 지원 / 재구현은 비권장, UI 래핑은 가능 |
| 수익화 | 구축 대행 · 한국어 교육 · 업종 특화 서비스 조합 권장 |

---

*본 문서는 저장소 내용을 직접 확인해 작성했습니다. 버전 및 기능은 업스트림 변경에 따라 달라질 수 있으니*
*최신 정보는 https://github.com/dokku/dokku 와 https://dokku.com 을 참고하세요.*
