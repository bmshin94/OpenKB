# OpenKB 전수조사 분석 정리 (한국어)

> 이 문서는 OpenKB 저장소를 전수조사하며 나눈 대화를 정리한 기록입니다.
> 작성일: 2026-09-20

## 관련 GitHub 주소

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/OpenKB |
| 원본 저장소 (upstream) | https://github.com/VectifyAI/OpenKB |
| 이슈 트래커 | https://github.com/VectifyAI/OpenKB/issues |
| PageIndex (핵심 검색 엔진) | https://github.com/VectifyAI/PageIndex |
| PageIndex MCP 서버 | https://github.com/VectifyAI/pageindex-mcp |
| ChatIndex | https://github.com/VectifyAI/ChatIndex |
| ConDB | https://github.com/VectifyAI/ConDB |
| PageIndex 문서 | https://docs.pageindex.ai/ |
| markitdown | https://github.com/microsoft/markitdown |
| LiteLLM | https://github.com/BerriAI/litellm |
| OpenAI Agents SDK | https://github.com/openai/openai-agents-python |

---

## 1. OpenKB란 무엇인가

**원시 문서를 LLM으로 컴파일해서 상호 연결된 위키형 지식베이스를 만들어 주는 오픈소스 CLI**입니다.

- 제작: VectifyAI (PageIndex 개발팀)
- 라이선스: Apache-2.0
- 언어: Python 3.10+ (엔진/CLI/REST) + React 19 + TypeScript (웹 UI)
- 개발 상태: Alpha (`Development Status :: 3 - Alpha`)
- 아이디어 출처: Andrej Karpathy가 제안한 "LLM이 요약·개념 페이지·상호 참조를 자동 유지하는 위키" 개념

### 전통적 RAG와의 차이

| 항목 | 전통적 RAG | OpenKB |
| --- | --- | --- |
| 질의 시 동작 | 매번 문서를 다시 검색 | 이미 컴파일된 위키를 읽음 |
| 지식 축적 | 누적되지 않음 | 문서를 넣을수록 위키가 성장 |
| 문서 간 연결 | 없음 | 개념 페이지가 문서를 교차 종합 |
| 벡터 DB | 필요 | 불필요 (vectorless) |
| 긴 문서 | 컨텍스트 한계 | PageIndex 트리 인덱스로 추론 검색 |

---

## 2. 저장소 구조 (전수조사 결과)

### 최상위

| 경로 | 설명 |
| --- | --- |
| `openkb/` | 본체 Python 패키지 (약 1.9만 줄) |
| `frontend/` | Knowledge Workbench 웹 UI (React 19 / Vite / Tailwind / shadcn-ui) |
| `skills/` | 에이전트 스킬 4종 (`openkb`, `openkb-deck-neon`, `openkb-deck-editorial`, `openkb-html-critic`) |
| `.claude-plugin/marketplace.json` | Claude Code 플러그인 마켓플레이스 매니페스트 |
| `examples/` | 실제 생성 결과 샘플 (위키/슬라이드/그래프/스킬) + 샘플 PDF |
| `tests/` | pytest 테스트 52개 파일 |
| `docs/golden-principles.md` | 에이전트가 지켜야 할 기계적 코딩 규칙 |
| `AGENTS.md` / `CLAUDE.md` | 에이전트용 저장소 지도 |
| `pyproject.toml` | 의존성 전부 정확 버전 고정 (공급망 보안) |
| `.github/workflows/` | CI(lint/type/test) 및 PyPI 퍼블리시 |

### `openkb/` 주요 모듈

| 모듈 | 역할 |
| --- | --- |
| `cli.py` | Click 기반 CLI 진입점 (3,808줄, tech-debt로 표기됨) |
| `config.py` | 설정 로드/검증, LiteLLM 패스스루 |
| `converter.py` | 문서 → 마크다운 변환 (markitdown) |
| `url_ingest.py` | URL 수집 (trafilatura) |
| `indexer.py` | 긴 문서용 PageIndex 트리 인덱싱 |
| `images.py` | 그림/이미지 추출 |
| `agent/compiler.py` | LLM 위키 컴파일러 (2,372줄, 핵심) |
| `agent/chat.py`, `agent/chat_session.py` | 위키 기반 멀티턴 채팅 + 세션 영속화 |
| `agent/query.py` | 단발 질의 생성기 |
| `agent/linter.py` | LLM 기반 의미 린트 (모순/공백/노후화) |
| `agent/tools.py` | 공유 위키 읽기/쓰기 툴 |
| `lint.py` | 구조 린트 (깨진 링크, 고아 페이지, 인덱스 동기화) |
| `locks.py` | 원자적 쓰기 / 파일 락 (portalocker) |
| `mutation.py` | 크래시 세이프 직렬 KB 변경 |
| `state.py` | 실행/세션 상태 추적 |
| `frontmatter.py` | YAML 프런트매터 왕복 처리 (OKF) |
| `schema.py` | 페이지/콘텐츠 스키마 상수 |
| `skill/` | Skill Factory (creator/validator/evaluator/marketplace/workspace) |
| `deck/` | 단일 파일 HTML 슬라이드 덱 생성 |
| `visualize.py`, `tree_renderer.py` | 지식 그래프 / 렌더링 |
| `watcher.py`, `watch_service.py` | `raw/` 폴더 감시 자동 컴파일 |
| `api.py` + `api_*.py` | FastAPI REST 서버 (엔드포인트 26개) |

### 생성되는 지식베이스 레이아웃

```
my-kb/
├── .openkb/config.yaml     # 모델/언어/임계값 등 설정
├── raw/                    # 원본 문서 보관
└── wiki/
    ├── AGENTS.md           # LLM이 런타임에 읽는 위키 작성 규칙
    ├── index.md            # 전체 목차 (자동 유지)
    ├── log.md              # append-only 작업 이력
    ├── sources/            # 변환된 원문 (.md 또는 페이지별 .json)
    ├── summaries/          # 문서당 요약 페이지
    ├── concepts/           # 문서 교차 개념 종합 페이지
    ├── entities/           # 인물/조직/장소/제품/작품/사건
    ├── explorations/       # 저장한 질의 결과
    └── reports/            # 린트 헬스체크 리포트
```

결과물은 전부 마크다운 + `[[위키링크]]` 이므로 Obsidian에서 그대로 열립니다.

---

## 3. 동작 파이프라인

```
openkb add <파일|폴더|URL>
  1) raw/ 복사 + 해시 등록(중복 방지)
  2) 짧은 문서 → markitdown 전문 변환
     긴 PDF(기본 20p 이상) → PageIndex 트리 인덱스
  3) 이미지/그림/표 추출
  4) LLM 컴파일러:
       - 요약 페이지 생성
       - 기존 concepts/ entities/ 읽기
       - 개념 페이지 신규 생성 또는 교차 병합
       - 엔티티 페이지 생성/갱신
       - index.md / log.md 갱신
  → 문서 1개가 보통 위키 페이지 10~15개에 영향
```

---

## 4. 명령어 정리

### Layer 1 — 위키 파운데이션

| 명령 | 설명 |
| --- | --- |
| `openkb init` | 지식베이스 초기화 (대화형) |
| `openkb add <경로/URL>` | 문서 추가 + 위키 컴파일 |
| `openkb list` | 문서/개념 목록 |
| `openkb status` | KB 경로 및 통계 |
| `openkb watch` | `raw/` 감시 후 자동 컴파일 |
| `openkb lint` | 구조 + 지식 건강검진 |
| `openkb remove <doc>` | 문서 및 파생 페이지 정리 제거 |
| `openkb recompile [doc] [--all]` | 재인덱싱 없이 컴파일 재실행 |
| `openkb use <path>` | 기본 KB 전환 |
| `openkb delete-kb <name>` | KB 삭제 |

### Layer 2 — 생성기(Generators)

| 명령 | 결과물 |
| --- | --- |
| `openkb query "질문"` | 출처가 붙은 근거 기반 답변 (`--save`로 보존) |
| `openkb chat` | 대화형 세션 (`--resume/--list/--delete`) |
| `openkb visualize` | 3D/마인드맵/방사형 인터랙티브 지식 그래프 HTML |
| `openkb skill new <name> "<intent>"` | 재배포 가능한 에이전트 스킬 폴더 |
| `openkb skill validate/eval/history/rollback` | 스킬 검증/트리거 평가/이력/롤백 |
| `openkb deck new <name> "<intent>"` | 단일 파일 HTML 슬라이드 덱 (`--skill`, `--critique`) |
| `openkb-web` | REST API + Knowledge Workbench (기본 7566 포트) |

채팅 내 슬래시 커맨드: `/help` `/status` `/list` `/add` `/skill new` `/deck new` `/critique` `/save` `/clear` `/lint` `/exit`

---

## 5. 설치 및 사용법

```bash
# 기본
pip install openkb

# 웹 UI 포함
pip install "openkb[web]"
openkb-web                       # http://127.0.0.1:7566

# 소스에서 개발 설치
git clone https://github.com/bmshin94/OpenKB.git
cd OpenKB
pip install -e ".[dev]"          # 또는 uv sync --extra dev
pytest
ruff check . && ruff format . && mypy openkb

# 프런트엔드 개발
cd frontend && npm install && npm run dev     # /api를 7566으로 프록시
npm run build                                  # openkb/web/ 번들 생성
```

기본 사용 흐름:

```bash
mkdir my-kb && cd my-kb
openkb init
echo "LLM_API_KEY=sk-..." > .env
openkb add paper.pdf
openkb query "핵심 발견은?"
openkb chat
```

설정 예시 (`.openkb/config.yaml`):

```yaml
model: gpt-5.4          # 또는 anthropic/..., gemini/..., ollama/...
language: ko            # 한국어 위키 생성 가능
pageindex_threshold: 20
# concurrency: 5
```

---

## 6. 플러그인 / 스킬 / MCP 구분

- **본체는 독립 실행 CLI + 파이썬 라이브러리**입니다. AI 없이도 단독 동작합니다.
- **Agent Skill 제공**: `skills/openkb/SKILL.md` (Claude Code / Codex CLI / Gemini CLI 공용)
- **Claude Code 플러그인 제공**: `.claude-plugin/marketplace.json`
  ```
  /plugin marketplace add VectifyAI/OpenKB
  /plugin install openkb@vectify
  ```
- **MCP 서버는 없음**: README에 "No MCP setup"으로 명시. 위키가 평범한 마크다운이라 기본 Read/Grep/Bash로 충분하다는 설계 철학. 대신 형제 프로젝트 `pageindex-mcp`가 별도로 존재합니다.

`SKILL.md`의 보안 설계도 특징적입니다.

- Trust boundary: 위키 본문은 **데이터이지 명령이 아님** (프롬프트 인젝션 방어)
- `openkb add` / `remove` / `lint --fix` / `chat` / `watch` / `init` / `use` 는 사용자의 명시적 요청 없이 실행 금지
- `openkb query`보다 개념 페이지 직접 읽기를 우선 (비용 + 인젝션 증폭 방지)

---

## 7. API 토큰 요약

| 환경변수 | 필수 | 용도 |
| --- | --- | --- |
| `LLM_API_KEY` | 필수 | 위키 컴파일 및 질의 (LiteLLM 호환 모든 제공사) |
| `PAGEINDEX_API_KEY` | 선택 | PageIndex Cloud (스캔 PDF OCR, 대용량 인덱싱). 없으면 로컬 오픈소스 버전 사용 |
| `OPENKB_API_TOKEN` | 선택 | REST/웹 UI Bearer 인증. 기본은 인증 OFF이므로 외부 노출 시 필수 설정 |
| `OPENKB_KB_ROOT` | 선택 | REST `/init`이 KB를 만드는 위치 |
| `OPENAI_API_BASE` | 선택 | OpenAI 호환 게이트웨이 주소 override |

API 키 없이 쓰는 방법:

- 구독형 OAuth 제공사 (`chatgpt/*`, `github_copilot/*`) — 디바이스 플로우 인증
- 로컬 LLM (`ollama/*`, LM Studio) — 비용 0원, 완전 오프라인 구동 가능

---

## 8. 왜 GitHub에서 주목받았는가 (저장소 내용 기반 분석)

1. Andrej Karpathy가 제안한 개념을 실제 구현한 첫 오픈소스라는 서사
2. "벡터 DB 없이 되는 RAG"라는 반(反)트렌드 메시지 (Trendshift 배지 노출)
3. 마크다운 위키 / 3D 그래프 / HTML 덱 등 **눈에 보이는 결과물**
4. Obsidian 호환 = 벤더 락인 없음, 기존 사용자층 흡수 용이
5. Skill Factory("책을 넣으면 디지털 전문가가 나온다")라는 강한 훅
6. 엔지니어링 신뢰도: 정확 버전 핀 고정, 원자적 쓰기/파일락, 모듈 800줄 제한을 테스트로 강제, `uv sync --locked` CI, 액션 커밋 해시 핀, 테스트 52개 파일
7. PageIndex / ChatIndex / ConDB / pageindex-mcp 생태계 크로스 프로모션

> 참고: 별(star) 수 등 실시간 수치는 확인하지 않았으며, 위 분석은 저장소 내용에 근거합니다.

---

## 9. 로컬 에이전트 구축에 주는 도움

바로 참고 가능한 패턴:

| 필요 기능 | 참고 파일 |
| --- | --- |
| 멀티 프로바이더 LLM 연결 | `config.py` (LiteLLM 패스스루) |
| 에이전트 툴 정의 | `agent/tools.py` |
| 멀티턴 세션 영속화 | `agent/chat_session.py` |
| 크래시 세이프 파일 쓰기 | `locks.py` |
| 직렬 트랜잭션 | `mutation.py` |
| 실행 상태 추적 | `state.py` |
| 스킬 생성/검증/평가/배포 | `skill/` 전체 |
| SSE 스트리밍 API | `api.py`, `api_helpers.py` |
| 파일 감시 데몬 | `watcher.py`, `watch_service.py` |

추가 가치:

- 위키 자체가 **파일 기반 장기 기억(long-term memory)** 역할을 하므로 별도 DB 없이 에이전트 메모리를 구성할 수 있습니다.
- `AGENTS.md`(지도) + `docs/golden-principles.md`(기계적 규칙) 조합은 "에이전트 친화적 저장소" 설계 레퍼런스로 재사용할 만합니다.

주의점:

- `cli.py`(3,808줄), `agent/compiler.py`(2,372줄)는 저장소 스스로 tech-debt로 인정한 거대 파일이므로 구조를 그대로 따라하지 말 것
- 완전 로컬 LLM 사용 시 컴파일 품질 저하 가능
- Alpha 단계이므로 API 변경 가능성 있음

---

## 10. React / PHP로 만들 수 있는가

- **React**: 이미 `frontend/`에 구현되어 있음 (React 19, TypeScript, Vite 7, Tailwind 3, shadcn-ui/Radix, react-router 7, i18next, mermaid, katex, recharts, motion). 새로 만들기보다 **커스터마이징**이 정답. API 클라이언트는 `frontend/src/api/*.ts`에 정리되어 있음.
- **PHP**: 엔진 포팅은 비권장 (PageIndex/markitdown/LiteLLM/Agents SDK 모두 파이썬 생태계). 대신 아래 구조 추천.

```
[PHP(Laravel): 프론트 / 회원 / 결제 / 관리자]
        ↓ HTTP REST
[openkb-web FastAPI :7566]  ← Python 엔진
        ↓
     wiki/ 마크다운 파일
```

주요 REST 엔드포인트:

```
GET  /api/v1/kbs            POST /api/v1/init       POST /api/v1/add
POST /api/v1/query          POST /api/v1/chat       POST /api/v1/list
POST /api/v1/status         POST /api/v1/lint       POST /api/v1/remove
POST /api/v1/recompile      GET  /api/v1/watch/events (SSE)
POST /api/v1/deck           POST /api/v1/skill      GET  /api/v1/skill/{name}/archive
GET/PATCH /api/v1/kb/config GET  /api/v1/meta
```

Swagger 문서는 `/docs`에서 제공되어 Postman 임포트가 가능합니다.
읽기 전용 뷰어는 위키가 마크다운이므로 PHP 단독 구현도 가능합니다.

---

## 11. 수익화 아이디어

### 라이선스 전제 (Apache-2.0)

- 상업적 이용 / 수정 / 클로즈드 소스 재배포 **가능**, 특허 사용권 명시적 부여
- 의무: LICENSE 사본 포함, 저작권 고지, **변경 사항 명시**
- 제한: "OpenKB", "PageIndex" 등 **상표는 사용 불가** → 자체 브랜드 필요

### 아이디어 목록

| # | 아이디어 | 난이도 | 수익성 | 요약 |
| --- | --- | --- | --- | --- |
| 1 | 버티컬 SaaS (법률/의료/제조/공공/교육) | 높음 | 매우 높음 | `wiki/AGENTS.md`만 바꿔도 도메인 특화 가능 |
| 2 | 데스크탑 앱 (Electron/Tauri) | 낮음 | 중상 | 설치·API키 진입장벽 제거, buy-once 또는 구독 |
| 3 | Skill 마켓플레이스 | 매우 높음 | 높음 | `openkb skill new` 산출물 유통, 수수료 20~30% |
| 4 | 사내 지식베이스 구축 컨설팅 | 낮음 | 높음 | 온프레미스 + 로컬 LLM = 보안 세일즈 포인트, 즉시 현금화 |
| 5 | 커넥터 애드온 (Notion/Slack/Drive/Confluence) | 중간 | 중상 | OpenKB가 로컬 파일만 먹는 빈틈 공략 |
| 6 | AI 슬라이드/리포트 생성 서비스 | 낮음 | 중상 | `deck new` + `html-critic` 활용, 출처 인용 자동 |
| 7 | 학술 특화 SaaS | 중간 | 중상 | arXiv/PubMed 연동, 연구 지도 + 연구 공백 탐지 |
| 8 | 관리형 호스팅 (Open-core) | 높음 | 높음 | 권한관리/감사로그/협업/SSO를 유료 기능으로 |
| 9 | 교육 콘텐츠·강의 | 낮음 | 중간 | "벡터DB 없는 RAG", 에이전트 아키텍처 실전 |
| 10 | 컴파일된 지식 상품 판매 | 낮음 | 중상 | 위키 자체를 판매 (공개 자료 기반, 저작권 주의) |

### 추천 로드맵

1. **Phase 1 (1~2개월)**: 컨설팅(#4)으로 매출 확보 + 콘텐츠(#9)로 브랜딩, 실제 니즈 파악
2. **Phase 2 (3~6개월)**: 반응 좋은 버티컬 1개 선정 → 버티컬 SaaS MVP(#1). 기존 React 프론트 재활용 + 인증/결제 추가
3. **Phase 3 (6~12개월)**: Open-core 요금제(#8), 생태계 성숙 시 Skill 마켓플레이스(#3)

### 리스크

| 리스크 | 대응 |
| --- | --- |
| 원본 팀의 직접 SaaS 진출 | 버티컬 특화 / 국내 시장 / 온프레미스로 차별화 |
| LLM API 원가 | 컴파일은 저가 모델, 질의는 고급 모델 / BYOK / 캐싱 |
| Alpha 단계 불안정성 | 버전 핀 고정, 업스트림 변경 모니터링 |
| Apache-2.0 고지 의무 | LICENSE 포함 + NOTICE에 변경사항 명시 |
| 상표 사용 제한 | 자체 브랜드명 사용 |
| 기본 인증 OFF | `OPENKB_API_TOKEN` 적용 + 테넌트 격리 필수 |

가장 현실적인 단기 전략은 **온프레미스 사내 지식베이스 구축(#4)** 입니다. 로컬 LLM으로 폐쇄망 구동이 가능하다는 점이 국내 기업 환경에서 강한 차별점이 됩니다.

---

## 12. 한 줄 결론

OpenKB는 "문서를 한 번 컴파일해 위키로 축적하고, 그 위키를 답변·채팅·그래프·슬라이드·에이전트 스킬로 재활용하는" 시스템이며, 로컬 에이전트 개발 레퍼런스이자 Apache-2.0 기반 상업화 토대로 모두 활용할 수 있습니다.
