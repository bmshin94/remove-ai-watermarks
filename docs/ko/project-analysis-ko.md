# remove-ai-watermarks 프로젝트 분석 (한국어)

이 문서는 `remove-ai-watermarks` 저장소를 전수조사한 결과와,
설치/사용법·배포 형태·유명세 분석·활용 방안·수익화 아이디어를 정리한 한국어 노트입니다.

- 작성일: 2026-09-20
- 분석 대상 커밋: `d97a2dd`
- 분석 범위: 파일 552개 / 약 486MB / `src` 파이썬 26,468줄

## 관련 GitHub 주소

| 대상 | 주소 |
|---|---|
| 원본 저장소 (upstream) | https://github.com/wiltodelta/remove-ai-watermarks |
| 이 저장소 (fork) | https://github.com/bmshin94/remove-ai-watermarks |
| ComfyUI 노드 패키지 | https://github.com/wiltodelta/ComfyUI-remove-ai-watermarks |
| PyPI 배포 | https://pypi.org/project/remove-ai-watermarks/ |
| 호스팅 서비스 (원저자 운영) | https://raiw.cc |
| 스폰서 | https://github.com/sponsors/wiltodelta |
| Agent Skills 스펙 | https://agentskills.io/specification |

> 원본 저장소는 2026-09-20 기준 스타 약 5,601개 / 포크 519개입니다.
> 이 저장소는 그 포크이며, `ce35f2c`에서 `CLAUDE.md`에 페르소나 가이드가 추가되었습니다.

---

## 1. 이 프로젝트는 무엇인가

**사용자가 직접 생성하거나 편집한 이미지·영상에서 AI 출처(provenance) 표식을 제거하는
Python CLI 도구이자 라이브러리**입니다. PyPI 배포명은 `remove-ai-watermarks`, 버전은 0.41.1,
라이선스는 Apache-2.0입니다.

### 다루는 표식 3종

| 종류 | 예시 | 육안 확인 | GPU |
|---|---|:---:|:---:|
| 가시 워터마크 | Gemini 반짝이, `豆包AI生成`, Sora/Veo/Kling 라벨 | 보임 | 불필요 |
| 비가시 워터마크 | SynthID, Microsoft InvisMark, Meta Content Seal | 안 보임 | CUDA 필수 |
| 메타데이터 | C2PA, EXIF, XMP, IPTC, 중국 TC260 AIGC 태그 | 파일 속성 | 불필요 |

### 가시 워터마크 등록 목록

이미지 13종 (`src/remove_ai_watermarks/watermark_registry.py`, 1,126줄):
`gemini`, `doubao`, `jimeng`, `jimeng_pill`, `qwen`, `kling`, `yuanbao`,
`samsung`, `runninghub`, `baidu`, `liblib`, `liblib_pill`, `microsoft`

영상 7종: `sora`, `veo`, `seedance`, `doubao`, `dola`, `hailuo`, `kling`

### 처리 방식

- **가시**: 예상 영역 탐지 → 마스크 생성 → 마스크 영역만 채우기(OpenCV / MI-GAN / LaMa).
  이미지 전체를 건드리지 않으므로 나머지 픽셀은 원본 그대로 유지됩니다.
- **비가시**: 디퓨전 파이프라인으로 이미지를 통째로 재생성합니다.
  기본 `qwen-zimage`(Qwen-Image-2512 Lightning + Canny ControlNet + SAM 마스크 얼굴 복구),
  대안으로 `sdxl-zimage`(Google 이미지 자동 선택), `chroma-zimage`(Microsoft 이미지 자동 선택).
  세 프로파일 모두 CUDA 전용이며 CPU/MPS 폴백이 없습니다.
- **메타데이터**: 포맷별 무손실 제거. JPEG는 인코딩된 scan을 재압축 없이 보존하고,
  MP4/MOV는 박스 크기와 미디어 오프셋을 유지한 채 값만 비웁니다.
  MKV/WebM/AVI/FLV는 스트림 복사로 리먹싱합니다.

---

## 2. 저장소 구조

```
src/remove_ai_watermarks/     본체 (.py 66개, 26,468줄)
├─ cli.py                     click 기반 CLI 엔트리포인트
├─ watermark_registry.py      13종 마크 좌표·실루엣 캘리브레이션
├─ *_engine.py (11개)         벤더별 탐지 엔진
├─ video_*.py (5개)           인코딩 / 시간축 일관성 / SynthID / 가시마크
├─ _internal/ (23개)          C2PA·ISOBMFF·EBML·RIFF·FLV 파서 자체 구현
│                             + 디퓨전 파이프라인 3종
├─ classify.py                픽셀 기반 AI/카메라 분류 (옵션, identify와 별개)
└─ source_classify.py         OpenAI/Google/unknown 출처 추정 (옵션)

docs/    (45개)               사용자 가이드 + 연구 아카이브
tests/   (126개)              문서-코드 일치 테스트 포함
data/    (약 460MB)           fixtures 91M / evaluations 83M / synthid 43M
scripts/ (50개+)              연구용 하네스 (제품 경로 아님)
skills/remove-ai-watermarks/  배포되는 Agent Skill
.claude-plugin/               Claude Code 플러그인 마켓플레이스 정의
.claude/rules/, .claude/skills/  개발자용 내부 규칙·스킬
.github/workflows/ (7개)      test / publish(PyPI) / distribute / HF / skill
```

### 핵심 문서

- `docs/index.md` — 문서 라우팅 허브
- `docs/cli.md`, `docs/installation.md`, `docs/python-api.md` — 사용자 가이드
- `docs/supported-signals.md` — 지원 경계
- `docs/known-limitations.md` — 한계 명시
- `docs/legal-and-safety.md` — 범위·안전·법적 맥락
- `docs/module-internals.md` — 모듈별 설계 결정·임계값·사고 기록
- `docs/agent-skill.md` — 스킬 설치·배포 전략

---

## 3. 범위 경계 (프로젝트의 생명선)

### 의도된 범위

- 가시 AI 생성 라벨 제거
- 비가시 출처 워터마크 제거
- C2PA·메타데이터 기반 AI 공시 제거
- 오탐 정정, 상호운용성 작업, 워터마크 견고성 연구

### 명시적 비범위

- 스톡 에이전시 미리보기
- 마켓플레이스·중고거래 마크
- 구매 게이트용 타일 오버레이
- 아티스트 보호 시스템(Nightshade, Glaze)

이 경계는 `README.md`, `CLAUDE.md`, `skills/remove-ai-watermarks/SKILL.md`,
`docs/legal-and-safety.md` 네 곳에 반복 명시되어 있습니다.
`erase --region`은 사용자가 직접 영역을 지정하는 범용 도구로 유지되며,
자동 스톡 워터마크 제거기로 확장하지 않는 것이 규칙입니다.

### 제거가 증명하지 않는 것

로컬 신호가 없다는 것은 "깨끗하다"가 아니라 "모른다"는 뜻입니다.
제거는 사람이 만들었음을 증명하지 않고, 서버 측 생성 이력을 지우지 않으며,
생성 계정을 익명화하지 않고, 모든 통계적 AI 검출기를 무력화하지 않습니다.

---

## 4. 설치 및 사용법

### 설치 (extras 방식)

```bash
# 기본 (메타데이터 전용, 최경량)
uv tool install remove-ai-watermarks

# 가시 워터마크 제거 (GPU 불필요)
uv tool install --force "remove-ai-watermarks[visible]"

# 영상 처리
uv tool install --force "remove-ai-watermarks[video]"

# 비가시 워터마크 제거 (NVIDIA CUDA 필수)
uv tool install --force "remove-ai-watermarks[qwen-zimage]"

# 전체
uv tool install --force "remove-ai-watermarks[all]"
```

`pipx`, `pip`도 동일한 extras 문법을 지원합니다.
Homebrew(`brew install wiltodelta/tap/remove-ai-watermarks`)는 formula가 extras를
담을 수 없어 메타데이터 계열 명령만 동작합니다.

요구사항: Python 3.11–3.14 / 영상은 ffmpeg / 비가시 제거는 NVIDIA CUDA.

### 주요 명령

```bash
remove-ai-watermarks identify image.png                 # 신호 확인 (--json 지원)
remove-ai-watermarks visible image.png -o clean.png     # 가시 마크 제거
remove-ai-watermarks visible image.png -o clean.png --backend lama
remove-ai-watermarks erase image.png --region x,y,w,h -o clean.png
remove-ai-watermarks metadata image.png --remove -o clean.png
remove-ai-watermarks invisible image.png -o clean.png --force
remove-ai-watermarks all image.png -o clean.png
remove-ai-watermarks batch ./images --mode visible
remove-ai-watermarks classify image.png

remove-ai-watermarks video identify input.mp4
remove-ai-watermarks video all input.mp4 -o clean.mp4
remove-ai-watermarks video visible kling.mp4 --mark kling -o out.mp4
remove-ai-watermarks video metadata input.mp4 --remove -o clean.mp4
remove-ai-watermarks video batch ./videos --mode all
```

주의: 이미지 `metadata`는 `-o` 생략 시 원본을 덮어쓰고,
영상 `video metadata`는 반대로 `<source>_clean`을 새로 만듭니다.

### Python API

```python
import remove_ai_watermarks as raiw

result, removed = raiw.remove_visible("watermarked.png", "clean.png")
report = raiw.remove_visible_detailed("watermarked.png", "clean.png")
print(report.status)  # cleaned | partial | unvalidated | no_watermark

complete = raiw.remove_all("in.png", "out.png")
images = raiw.remove_batch("images", "images_clean", mode="visible")
video = raiw.remove_video_all("in.mp4", "out.mp4")
```

경로뿐 아니라 BGR NumPy 배열도 입력으로 받습니다.

### 개발 환경

```bash
uv sync --frozen --extra dev
bash maintain.sh
```

`maintain.sh`는 의존성 최신성(`uv-outdated`), 보안 스캔(`uv-secure`),
C2PA 소프트 바인딩 동기화 체크, Ruff, `src/` 범위 Pyright, 병렬 pytest를 순서대로 실행합니다.

---

## 5. 플러그인 / 스킬 / MCP 구분

본체는 **Python CLI + 라이브러리**이며, 그 위에 여러 배포 형태가 얹혀 있습니다.

| 형태 | 제공 | 설명 |
|---|:---:|---|
| CLI 도구 | O (본체) | PyPI `remove-ai-watermarks` |
| Python 라이브러리 | O (본체) | `import remove_ai_watermarks` |
| Agent Skill | O | `skills/remove-ai-watermarks/SKILL.md` — CLI 사용 매뉴얼 |
| Claude Code 플러그인 | O | `.claude-plugin/marketplace.json` v1.0.9 |
| MCP 서버 | **X** | 저장소에 MCP 관련 코드 없음 |
| ComfyUI 노드 | O | 별도 저장소 |
| 웹 서비스 | O | raiw.cc |

설치 명령:

```bash
npx skills add wiltodelta/remove-ai-watermarks
```

```text
/plugin marketplace add wiltodelta/remove-ai-watermarks
/plugin install remove-ai-watermarks@remove-ai-watermarks
```

Agent Skill은 실행 코드가 아니라 문서입니다.
에이전트가 `SKILL.md`를 읽고 CLI를 대신 호출하는 구조이며,
`scripts/probe.py`로 환경(파이썬, GPU, 설치도구, ffmpeg)을 먼저 검사하도록 지시합니다.

---

## 6. API 토큰 필요 여부

**기본 기능에는 토큰이 전혀 필요 없습니다.** `.env.example`의 항목은 모두 주석 처리된 선택 사항입니다.

| 변수 | 필수 | 용도 |
|---|:---:|---|
| `HF_TOKEN` | 선택 | 게이트/비공개 모델 접근 시 |
| `OPENAI_API_KEY` | 선택 | 개발용 `scripts/openai_provenance_check.py` 전용 |
| `AZURE_CONTENT_SAFETY_*` | 선택 | Microsoft 오라클 검증 연구용 |
| `THORDATA_ROUTE_*` | 선택 | 벤더 웹 오라클 프록시 |
| `RAIW_CLASSIFY_WEIGHTS` | 선택 | 분류기 가중치 로컬 경로 |

`identify` / `visible` / `erase` / `metadata` / `video` / `batch`는 토큰 없이 완전 오프라인 실행됩니다.
`invisible`과 `classify`도 토큰이 아니라 모델 다운로드만 필요합니다.
이미지가 외부 서버로 전송되지 않는 구조이므로 B2B·온프레미스 영업 포인트가 됩니다.

---

## 7. 왜 GitHub에서 주목받았는가

1. **타이밍** — SynthID, C2PA, 중국 TC260 AIGC 라벨 의무화가 겹친 시점에 등장
2. **키워드 선점** — `ai-watermark-remover`, `synthid`, `c2pa`, `nano-banana`,
   `gemini-watermark` 등을 `pyproject.toml` keywords에 배치
3. **빈 시장** — 탐지 도구는 많지만 제거 도구는 법적 부담 때문에 드물었음
4. **명확한 경계** — 제3자 유료 자산 워터마크를 코드·문서 레벨에서 거부해 존속 가능
5. **품질** — 소스 26,468줄 대비 테스트 126개 파일, 문서 45개,
   검증 날짜·방법·해시까지 기록된 연구 로그
6. **낮은 진입장벽** — GPU 없이도 가시 마크와 메타데이터 제거 가능
7. **다채널 유통** — PyPI, Homebrew, ComfyUI 레지스트리, Hugging Face Space,
   Agent Skill 카탈로그, Claude 플러그인, 웹 SaaS, GitHub Sponsors

---

## 8. 로컬 에이전트 구축에 참고할 패턴

- **3단 컨텍스트 계층**: `CLAUDE.md`(항상 로딩, 가볍게) → `.claude/rules/*.md`(해당 파일
  변경 시 자동 로딩) → `docs/*.md`(필요 시 읽기). 컨텍스트 윈도우 절약 구조.
- **probe 우선 원칙**: 에이전트가 환경을 추측하지 않고 `scripts/probe.py` 결과를 신뢰하도록 지시.
- **SKILL.md description 작성법**: "언제 쓰는지 + 언제 쓰면 안 되는지"를 한 문단에 압축해
  스킬 발동 조건을 명확히 함.
- **문서-코드 일치 테스트**: `tests/test_docs_cover_the_public_surface.py`,
  `tests/test_agent_skill.py`가 문서와 공개 surface의 불일치를 CI에서 차단.
- **단일 게이트 스크립트**: `maintain.sh` 하나로 의존성·보안·린트·타입·테스트를 일괄 실행.
- **거부 경계 설계**: 안전 정책이 있는 에이전트를 만들 때 참고할 구체적 사례.

---

## 9. 수익화 아이디어

### 검증된 선례

원저자는 raiw.cc에서 프리미엄 모델을 운영 중입니다.
무료는 가시 마크 + 메타데이터 제거를 Standard 출력 12MP까지 제공하고,
원본 해상도 12MP 초과와 비가시 워터마크 제거는 유료입니다.

원가 구조가 기술적으로 깔끔하게 분리되는 것이 핵심입니다.

- 가시 마크 + 메타데이터: CPU만 사용 → 원가 거의 0 → 무료로 개방 가능
- 비가시 워터마크: GPU 필수 → 건당 원가 발생 → 유료 티어 전용

### 아이디어 목록

| # | 아이디어 | 수익성 | 난이도 | 초기비용 | 비고 |
|---|---|:---:|:---:|:---:|---|
| 1 | 한국 시장 특화 웹 SaaS | 높음 | 높음 | 중 | raiw.cc는 영어권 타겟, 국내 경쟁 희박 |
| 2 | MCP 서버 오픈소스 | 낮음 | 낮음 | 0 | 현재 존재하지 않는 빈자리, 브랜딩 효과 |
| 3 | 플랫폼 플러그인 | 중 | 중 | 낮음 | WordPress, Chrome, Figma, Photoshop |
| 4 | B2B 온프레미스 라이선스 | 매우 높음 | 중 | 낮음 | "데이터 외부 반출 없음"이 셀링포인트 |
| 5 | API 종량제 판매 | 중 | 중 | 중 | RapidAPI, n8n/Zapier/Make 커넥터 |
| 6 | 콘텐츠·교육 | 낮음 | 낮음 | 0 | 한국어 콘텐츠 공백, SaaS 유입 채널 |

### 1) 한국 시장 특화 SaaS

타겟: 블로거/애드센스, 쇼핑몰 셀러, 유튜브 썸네일 제작자, 마케팅 대행사, 웹툰·일러스트 작가.

가격 예시:

```
무료        일 5장, 가시+메타, 최대 2MP
라이트      월 9,900원   월 300장, 원본 해상도, LaMa 백엔드
프로        월 29,000원  대량 처리, 영상 10분/월, API 100콜
비즈니스    월 99,000원  팀 5인, API 무제한, 비가시 제거, 우선 처리
엔터프라이즈 별도 견적     온프레미스, SLA, 전용 지원
```

전환 퍼널: 브라우저에서만 동작하는 "업로드 없는 EXIF 클리너"로 SEO 트래픽 확보 →
가시 워터마크 제거로 가입 유도 → 원본 해상도·비가시 제거로 결제 전환.

기술 스택 제안: Next.js(SSR/SEO) + Laravel 또는 FastAPI + Python 워커.
GPU는 상시 서버 대신 Runpod/Vast.ai 서버리스로 사용량 과금.
스토리지는 egress 무료인 Cloudflare R2, 결제는 토스페이먼츠/포트원.

### 2) MCP 서버

저장소에 MCP 서버가 없다는 점이 기회입니다.
직접 수익은 작지만 "AI 워터마크 = 나" 브랜딩과 SaaS 유입을 만드는 마케팅 자산이 됩니다.
Apache-2.0이므로 래핑·재배포가 합법입니다.

### 3) 플랫폼 플러그인

WordPress가 1순위입니다. 전 세계 웹사이트의 상당수가 WordPress이고,
"미디어 업로드 시 AI 메타데이터 자동 제거"는 설치 후 방치형이라 갱신율이 높습니다.
무료(메타데이터) / Pro 연 $49(가시 마크 + 대량 처리) 구성이 무난합니다.
Chrome 확장("이미지 우클릭 → 워터마크 제거 후 저장")은 진입장벽이 낮아 바이럴에 유리합니다.
ComfyUI 노드는 이미 공식 패키지가 있으므로 중복 개발을 피합니다.

### 4) B2B 온프레미스

토큰이 필요 없고 외부 통신이 없다는 점이 결정적 차별점입니다.
타겟은 광고·마케팅 대행사, 언론사, 대형 이커머스, 게임사, 망분리 공공기관.

```
스타터      연 300만원    1서버, 이메일 지원
스탠다드    연 800만원    3서버, 배치 파이프라인 구축, 전화 지원
엔터프라이즈 연 2,000만원+ 무제한, 커스텀 마크 등록, SLA, 온사이트
```

추가 수익원으로 "고객사 전용 워터마크 등록" 개발 용역(건당 수백만원)이 있습니다.
`watermark_registry.py` 구조가 재사용 가능하게 짜여 있어 한 번 만든 실루엣을 다른 고객에게도 적용할 수 있습니다.

### 5) API 종량제

```
가시 워터마크 제거   장당 10원
메타데이터 제거     장당 2원
영상 처리          분당 100원
비가시 워터마크     장당 200원
월 정액            5만원(1만 콜) / 20만원(5만 콜) / 50만원(대량)
```

RapidAPI 등록으로 글로벌 개발자에게 노출하고,
n8n/Zapier/Make 커넥터로 노코드 시장을 공략합니다.

### 6) 콘텐츠·교육

"AI 워터마크·프로버넌스" 주제는 한국어 콘텐츠가 거의 없어 선점 여지가 큽니다.
유튜브·블로그(애드센스 + SaaS 유입), 인프런/클래스101 강의, 전자책, 뉴스레터 스폰서십.

### 추천 로드맵

```
1단계 (1~2개월, 자본 0원)
  MCP 서버 오픈소스 공개 + 한국어 콘텐츠 시작 + 무료 EXIF 클리너(SEO 미끼)
  목표: 분야 브랜딩과 트래픽 확보

2단계 (3~6개월)
  한국어 SaaS MVP 런칭 + WordPress 플러그인 + API 공개
  목표: 월 100~300만원

3단계 (6개월~)
  B2B 온프레미스 영업 + 커스텀 마크 등록 용역 + 강의
  목표: 월 1,000만원+
```

---

## 10. React / PHP 구현 가능성

| 기능 | React | PHP | 비고 |
|---|:---:|:---:|---|
| 메타데이터 제거 | 가능 | 매우 쉬움 | `piexifjs`, `exiftool` 래퍼 |
| 영역 지정 제거(erase) | 가능 | 가능 | Canvas + inpainting 직접 구현 필요 |
| 가시 워터마크 탐지 | 어려움 | 어려움 | 13종 실루엣 캘리브레이션이 핵심 자산 |
| 비가시 워터마크 제거 | 불가 | 불가 | 디퓨전 + CUDA 필요 |
| 영상 처리 | 불가 | 가능 | PHP는 ffmpeg 호출 가능 |

권장 아키텍처는 재구현이 아니라 래핑입니다.

```
React 프론트  →  PHP/Laravel API 게이트웨이  →  Python 워커
(업로드 UI,      (인증, 결제, 파일 검증,        (remove-ai-watermarks,
 Before/After,    작업 큐, 사용량 미터링)        CPU/GPU 분리)
 영역 선택,
 진행률)
```

PHP에서 호출하는 두 가지 방법:

```php
// 1) CLI 직접 호출
$process = new Symfony\Component\Process\Process([
    'remove-ai-watermarks', 'visible', $input, '-o', $output, '--backend', 'lama',
]);
$process->run();

// 2) FastAPI 마이크로서비스 (권장)
$res = Http::attach('file', file_get_contents($path), 'image.png')
           ->post('http://python-worker:8000/visible');
```

메타데이터 제거만 필요하다면 서버 없이 브라우저에서 100% 처리할 수 있습니다.
파일이 업로드되지 않으므로 서버 비용이 0이고 프라이버시 측면에서도 강력한 무료 미끼 상품이 됩니다.

26,468줄과 워터마크 캘리브레이션 데이터를 React/PHP로 전면 포팅하는 것은 권장하지 않습니다.
Apache-2.0 라이선스이므로 그대로 호출해 쓰는 것이 합법이며 유지보수 측면에서도 유리합니다.

---

## 11. 상업화 시 준수 사항

1. **범위 경계 유지** — 제3자 유료 자산 워터마크는 서비스 범위에서 제외하고,
   랜딩 페이지·이용약관·가입 동의에 명시합니다. 이를 어기면 결제 대행 차단,
   법적 리스크, 앱스토어 퇴출로 이어질 수 있습니다.
2. **라이선스 준수** — Apache-2.0은 상업적 이용·수정·재배포를 허용하지만
   라이선스 사본 포함, 저작권 고지(Copyright 2025-2026 wiltodelta) 유지,
   변경 사항 명시가 조건입니다. 디퓨전 모델 가중치는 각각의 라이선스를 별도로 확인해야 합니다.
3. **정직한 마케팅** — "100% 완벽 제거 보장" 같은 표현은 사용하지 않습니다.
   원본 README도 모든 검증기가 결과물을 거부한다고 보장하지 않는다고 명시합니다.
   지원 범위와 결과 편차를 그대로 안내하는 편이 법적으로도 안전합니다.

---

## 12. 알려진 한계 요약

- 로컬 신호 부재는 "깨끗함"이 아니라 "알 수 없음"입니다.
- 가시 제거는 작은 영역을 재구성하므로 배경과 백엔드에 따라 품질이 달라집니다.
  OpenCV 백엔드는 구조적 배경에서 번질 수 있어 MI-GAN 또는 LaMa가 유리합니다.
- 비가시 제거는 이미지 전체를 변경하며 얼굴·글자·미세 디테일이 달라질 수 있습니다.
- 영상 SynthID 재생성은 해상도, 프레임레이트, 디테일을 변경합니다.
  공개 로컬 디코더가 없어 런타임에 임의 출력의 제거 여부를 확정할 수 없습니다.
- 비가시 제거(CLI/고수준 API)는 CUDA 전용이며 다른 디바이스에서는 생성 단계에서 거부됩니다.
- 벤더 워터마크 체계는 변경될 수 있으므로 중요한 결과물은 제공자의 검증기로 재확인해야 합니다.
