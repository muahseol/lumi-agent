# 루미(Lumi) · 버추얼 아이돌 AI 에이전트 — Starter Code

> AI Backend Engineering (Service Deployment, LLMOps) 실습용 스타터 코드

팬들의 "덕질"을 도와주는 버추얼 아이돌 **루미(Lumi)** AI 에이전트 서비스입니다.
대화, 스케줄·프로필 정보 제공, 캘린더 등록·팬레터 저장 같은 액션을 수행하는 에이전트를
**FastAPI + LangGraph + Upstage Solar + Supabase(pgvector)** 로 단계별로 구현해 나갑니다.

이 저장소는 **출발점(starter)** 입니다. 여기서부터 실습을 따라가며 **LangGraph 에이전트**(state·nodes·edges·graph), **RAG 문서 검색**, **툴(Tool) 실행**, **SSE 스트리밍**, **CI 파이프라인**, **Docker 배포**까지 단계별로 직접 구현해 나갑니다.

---

## ✅ 사전 준비물 (Prerequisites)

- **Python 3.11 이상**
- **[uv](https://docs.astral.sh/uv/)** (파이썬 패키지·가상환경 매니저)
  ```bash
  # macOS / Linux
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
- **Upstage API 키** — [https://console.upstage.ai/](https://console.upstage.ai/) 에서 발급
- **Supabase 프로젝트** — [https://supabase.com/](https://supabase.com/) 에서 생성 (URL + API Key) · *RAG 실습 시 필요*

---

## 🚀 빠른 시작 (Quick Start)

### 1. 의존성 설치

```bash
uv sync
```

> `uv` 가 `pyproject.toml` / `uv.lock` 을 읽어 `.venv/` 가상환경을 자동으로 만들고
> 필요한 패키지를 설치합니다. (`requires-python >=3.11`)

> 💡 **개발·실습 도구 설치**
>
> ```bash
> uv sync --extra dev      # 테스트·린트 (CI 실습시 활용: pytest·ruff) — CI도 이 명령을 사용
> uv sync --all-extras     # 위 + Jupyter 노트북까지 한 번에 (study/ 노트북 실행용)
> ```
>
> `study/` 참고 노트북을 열려면 Jupyter가 필요합니다 (`notebook` extra).
> 노트북만 추가하려면 `uv sync --extra notebook`, 노트북 열기: `uv run jupyter lab`

### 2. 환경변수 설정

`.env.example` 을 복사해 `.env` 를 만들고 본인의 키를 채웁니다.

```bash
# macOS / Linux
cp .env.example .env

# Windows (CMD / PowerShell)
copy .env.example .env
```

```dotenv
# .env
ENVIRONMENT=development
DEBUG=true

# Upstage Solar API 키 (필수)
UPSTAGE_API_KEY=발급받은_본인의_키
LLM_MODEL=solar-pro3

# Supabase 설정 (RAG 실습 시 필요)
SUPABASE_URL=https://<your-project-ref>.supabase.co
SUPABASE_KEY=발급받은_본인의_키
```

> ⚠️ `.env` 에는 **본인의 키만** 넣으세요. `.env` 는 절대 공유·커밋하지 않습니다(`.gitignore` 에 등록되어 있음).

### 3. 서버 실행

```bash
# 개발 서버 (코드 변경 시 자동 리로드)
uv run uvicorn app.main:app --reload
```

서버가 뜨면 아래 주소로 접속합니다.

| 항목                  | URL                                                     |
| --------------------- | ------------------------------------------------------- |
| 루트                  | [http://localhost:8000/](http://localhost:8000/)           |
| Swagger UI (API 문서) | [http://localhost:8000/docs](http://localhost:8000/docs)   |
| ReDoc                 | [http://localhost:8000/redoc](http://localhost:8000/redoc) |

정상 동작 확인:

```bash
curl http://localhost:8000/
# {"message":"Lumi Agent API에 오신 것을 환영합니다!", ...}
```

---

## ⚙️ 환경변수

| 변수                | 설명                                                                  | 기본값          |
| ------------------- | --------------------------------------------------------------------- | --------------- |
| `ENVIRONMENT`     | 실행 환경 (`development` / `staging` / `production` / `test`) | `development` |
| `DEBUG`           | 디버그 모드 (개발 시`true`)                                         | `true`        |
| `UPSTAGE_API_KEY` | Upstage Solar API 키                                                  | —              |
| `LLM_MODEL`       | 사용할 Solar 모델명                                                   | `solar-pro3`  |
| `SUPABASE_URL`    | Supabase 프로젝트 URL                                                 | —              |
| `SUPABASE_KEY`    | Supabase API 키                                                       | —              |

설정은 `app/core/config.py` 의 `Settings` 클래스에서 타입 안전하게 로드됩니다.

---

## 📁 프로젝트 구조

```
starter_code/
├── app/                       # 애플리케이션 코드
│   ├── main.py                # FastAPI 진입점 (앱 생성·미들웨어·라우터 등록)
│   ├── ui.py                  # Gradio 채팅 UI (`/ui` 에 마운트) — 제공됨
│   ├── core/
│   │   └── config.py          # 환경변수 설정 (pydantic-settings)
│   ├── schemas/
│   │   └── chat.py            # 채팅 요청/응답 스키마
│   └── static/                # UI 정적 파일 (파비콘·OG 이미지)
│       ├── favicon.svg
│       └── og-image.png
├── data/                      # 데이터 & 적재 스크립트
│   ├── supabase_schema.sql    # Supabase 테이블·pgvector·검색 함수 정의
│   ├── lumi_worldview_v2.5.md / .pdf   # 루미 세계관 문서 (active, RAG 원본)
│   ├── lumi_worldview_v1.0.md / .pdf   # 구버전 (deprecated, 메타필터용 distractor)
│   └── scripts/
│       ├── ingest_data.py     # 스케줄 샘플 데이터 적재
│       └── ingest_rag.py      # 세계관 문서 → 벡터 DB 적재 (RAG)
├── scripts/
│   └── ai_reviewer.py         # CI용 AI 코드 리뷰 스크립트 (CI 실습시 활용)
├── study/
│   └── langgraph-개념정리.ipynb   # LangGraph 개념 참고 노트북 (읽기용·실습 X)
├── tests/                     # pytest 테스트 (CI 실습시 활용)
│   ├── test_agent.py
│   └── test_api.py
├── .env.example               # 환경변수 템플릿
├── .pre-commit-config.yaml    # pre-commit 훅 (ruff 등) — CI 실습시 활용
├── pyproject.toml             # 의존성·도구 설정
└── uv.lock                    # 잠금 파일 (재현 가능한 설치)
```

> 위는 **제공되는 출발점**이고, 실습을 진행하며 `app/graph/`(state·nodes·edges·graph) ·
> `app/api/`(routes) · `app/tools/` · `app/repositories/`, 그리고 `Dockerfile` ·
> `docker-compose.yml` · `.github/workflows/`(CI·CD) 등을 **직접 추가·구현**합니다.

---

## 🗄️ 데이터 준비 (Supabase + RAG)

> RAG·스케줄 조회 실습 단계에서 진행합니다. 기본 서버 실행만 할 때는 건너뛰어도 됩니다.

### 1) 테이블·확장·함수 생성

Supabase 대시보드 → **SQL Editor** 에서 [`data/supabase_schema.sql`](data/supabase_schema.sql) 의 내용을 실행합니다.
`schedules`, `fan_letters`, `documents` 테이블과 `pgvector` 확장, `match_documents()` 검색 함수가 생성됩니다.

### 2) 스케줄 샘플 데이터 적재

```bash
uv run python data/scripts/ingest_data.py
```

### 3) 세계관 문서 적재 (RAG)

```bash
# 기본: v2.5(active) + v1.0(deprecated) 모두 적재
uv run python data/scripts/ingest_rag.py

# v2.5(active)만 적재
uv run python data/scripts/ingest_rag.py --active-only
```

> 이 스크립트는 **멱등성**을 보장합니다 — 실행할 때마다 기존 `documents` 를 비우고 새로 적재합니다.
> `v1.0(deprecated)` 문서는 메타데이터 필터링을 시연하기 위한 **Distractor(방해 문서)** 입니다.

---

## 📌 참고

- **API 문서**: 서버 실행 후 `/docs`(Swagger) · `/redoc` 에서 인터랙티브하게 확인할 수 있습니다.
- **Upstage Solar**: [https://console.upstage.ai/](https://console.upstage.ai/)
- **Supabase pgvector**: [https://supabase.com/docs/guides/database/extensions/pgvector](https://supabase.com/docs/guides/database/extensions/pgvector)
- **LangGraph**: [https://langchain-ai.github.io/langgraph/](https://langchain-ai.github.io/langgraph/)
- **FastAPI**: [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)

---

## 강의 노트
- 1강 핵심 목표 : StartCode의 핵심 구조 이해


### TODO - 1강
- [v] 환경 세팅
    - [v] Code Class Live Share 익스텐션 설치
        - [v] 화면 공유 테스트
    - [v] 가상 환경 생성
        - uv sync --all-extras
    - [v] .env 환경 변수 설정
    - [v] Supabase 설정
        - [v] 프로젝트에 data/supabase_schema.sql 실행
        - [v] 스케줄 데이터 저장
            - uv run python data/scripts/ingest_data.py
        - [v] RAG 데이터 저장
            - uv run python data/scripts/ingest_rag.py
    - [v] 프로젝트 실행
        - uv run uvicorn app.main:app --reload

- [v] StartCode 구조 이해
    - [v] 1강 핵심 코드 이해
        ├── app/
        ├── [v] main.py                # FastAPI 진입점 (앱 생성·미들웨어·라우터 등록)
        ├── core/
        │   └── [v] config.py          # 환경변수 설정
        ├── schemas/
            └── [v] chat.py            # 채팅 요청/응답 스키마
    - [v] 기타 코드
        - [v] data/ — Supabase 스키마 · 세계관 문서 · 데이터 적재 스크립트 (스케줄 및 RAG용 루미 프로필 문서)
        - app/ui.py — Gradio 채팅 UI (다음 시간 활용)
        - scripts/ — CI용 AI 코드 리뷰 스크립트 (내일 CI 실습시 활용)
        - tests/ — pytest 스펙 테스트 (내일 CD 실습시 활용)


# 강의 노트 2강
- 2강의 핵심 목표 : LangGraph 이용해서 에이전트 MVP 구현


### 프로젝트 구조 (2강 완성 목표)

```
app/
├── core/
│   ├── config.py          ← 설정 관리 (pydantic-settings)
│   └── prompts.py         ← 프롬프트 분리
├── schemas/
│   └── chat.py            ← API 요청/응답 모델
├── graph/                 ← 노트북에서 만든 에이전트
│   ├── state.py           ← State 정의
│   ├── nodes.py           ← 노드 구현
│   ├── edges.py           ← 라우팅 로직
│   └── graph.py           ← 그래프 조립 + 싱글톤
├── repositories/          ← DB 접근 계층
│   ├── rag.py             ← RAG 검색 (Supabase pgvector)
│   ├── schedule.py        ← 스케줄 조회
│   └── fan_letter.py      ← 팬레터 저장
├── tools/
│   └── executor.py        ← Tool 실행 (Repository 활용)
├── api/routes/
│   └── chat.py            ← REST API 엔드포인트
├── ui.py                  ← Gradio UI (/ui)
└── main.py                ← FastAPI 앱 (lifespan, CORS)
```


### TODO 정리
- [v] graph 구현 : Agent 핵심 로직 (노트북 코드 가져와서 필요한 부분 수정)
    - [v] graph/state.py 구현 : 노드가 공유하는 데이터
    - [v] graph/nodes.py 구현 : 실제로 작업을 수행하는 함수
        - [v] router 노드 : 메세지 의도분류
            - [v] core/prompts.py : router 프롬프트 구현
        - [v] rag 노드 : 문서 검색
            - [v] repositories 폴더 생성
                - [v] repositories/__init__.py : DB연결 구현
                - [v] repositories/rag.py : 루미 문서 정보 검색
        - [v] tool 노드 : 툴 실행
            - [v] tools/executor.py 구현 : tool 실행 로직
            - [v] repositories/schedule.py : 루미 스케쥴 데이터 조회
            - [v] repositories/fan_letter.py : 팬들의 메시지 저장
        - [] response 노드 : 루미 응답
            - [] core/prompts.py 구현
    - [] edges.py 구현 : 조건부 엣지 구현
    - [] graph.py 구현 : 그래프 빌드 (싱글톤)
- [] API 서버 구현
    - [] api/routes/chat.py : 엔드포인트에 따라 서비스 제공 (루미 챗봇)
- [] 프론트엔드 구현
    - [] ui.py : Gradio 사용 (이미 구현되어 있음 <------- 코드 교체!)
- [] main.py : lifespan에서 graph 싱글톤 로드 + 채팅 라우터 등록
