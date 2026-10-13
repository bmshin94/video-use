# video-use 전수조사 & 활용 전략 (한국어 정리)

> 이 문서는 `video-use` 저장소를 전수조사하고, 설치/사용법·정체성(스킬 vs MCP vs 플러그인)·API 요구사항·AI 에이전트 설계 관점의 가치·수익화 전략·React/PHP 이식 가능성·유튜브 콘텐츠화 가능성까지 한국어로 정리한 자료입니다.

## 저장소 주소

| 구분 | URL |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/video-use |
| 원본 (업스트림) | https://github.com/browser-use/video-use |
| 라이선스 | MIT (상업적 이용·수정·재배포 가능) |

**관련 링크**

- ElevenLabs API 키 발급: https://elevenlabs.io/app/settings/api-keys
- Browser Use Cloud: https://cloud.browser-use.com/v4
- Browser Use Box: https://browser-use.com/bux
- HyperFrames: https://github.com/heygen-com/hyperframes
- Remotion: https://www.remotion.dev/
- Manim Community: https://www.manim.community/

---

## 1. 한 줄 정의

> **Claude Code(터미널 AI 에이전트)와 대화만으로 영상 편집을 끝내는 오픈소스 "스킬"**

원본 영상 폴더에 날영상을 넣고 에이전트를 실행해 `"이거 런칭 영상으로 편집해줘"` 한 줄을 말하면
`<영상폴더>/edit/final.mp4` 가 산출됩니다. 프리셋·메뉴 없이 전부 자연어 대화로 진행됩니다.

이 포크에는 원본 외에 `CLAUDE.md`(페르소나 가이드, PR #1)가 추가되어 있습니다.

---

## 2. 저장소 전수조사 결과

```
video-use/
├── README.md               5.6KB   소개 + 설치 안내 + 작동 원리
├── install.md              7.7KB   AI가 읽고 스스로 설치하는 설치 지시서
├── SKILL.md               23.0KB   핵심 — AI 편집자의 작업 지침서
├── CLAUDE.md               1.4KB   페르소나 가이드 (이 포크에서 추가)
├── pyproject.toml           450B   의존성: requests, librosa, matplotlib, pillow, numpy
├── .env.example              20B   ELEVENLABS_API_KEY=
├── LICENSE                          MIT
├── poster.html            20.0KB   "How an LLM Understands Video" 인포그래픽 (Tailwind CDN)
│
├── helpers/                        파이썬 엔진 6개 / 총 2,111줄
│   ├── render.py            771줄  렌더링 총괄 (추출→병합→오버레이→자막→라우드니스)
│   ├── grade.py             375줄  색보정 (프리셋 + 데이터 기반 자동 보정)
│   ├── timeline_view.py     392줄  필름스트립 + 음파 + 단어라벨 PNG 합성
│   ├── transcribe.py        241줄  ElevenLabs Scribe 단일 전사 (캐싱/무음검사)
│   ├── pack_transcripts.py  206줄  전사 JSON → 구 단위 마크다운 압축
│   └── transcribe_batch.py  126줄  4워커 병렬 전사
│
├── skills/manim-video/             중첩 스킬 — 3Blue1Brown 스타일 수학 애니메이션
│   ├── SKILL.md + README.md
│   ├── references/                 14개 심화 문서 (수식/3D/그래프/디자인/트러블슈팅 등)
│   └── scripts/setup.sh
│
├── static/
│   ├── video-use-banner.png  512KB
│   └── timeline-view.svg      12KB  작동 원리 다이어그램
│
└── tests/
    ├── test_render_fps.py        143줄  프레임레이트 보존 테스트
    └── test_render_orientation.py 62줄  세로영상 회전 테스트
```

---

## 3. 핵심 아이디어 — "LLM은 영상을 보지 않고 읽는다"

| 접근 | 토큰 소모 |
|---|---|
| 순진한 방법: 프레임 30,000장 × 1,500토큰 | 약 4,500만 토큰 |
| video-use: 텍스트 12KB + PNG 몇 장 | 약 1만 토큰 |

약 **2,700배 압축**. 그런데 의사결정 능력은 유지합니다.

### Layer 1 — 오디오 전사 (항상 로드)

ElevenLabs Scribe(`scribe_v1`)를 소스당 1회 호출하여 다음을 획득합니다.

- 단어 단위 타임스탬프 (`timestamps_granularity=word`)
- 화자 분리 (`diarize=true`)
- 오디오 이벤트 태그 (`tag_audio_events=true`) — `(laughter)`, `(applause)`, `(sigh)`

모든 테이크를 하나의 `takes_packed.md`(약 12KB)로 압축한 것이 LLM의 주 열람 뷰입니다.

```
## C0103  (duration: 43.0s, 8 phrases)
  [002.52-005.36] S0 Ninety percent of what a web agent does is completely wasted.
  [006.08-006.74] S0 We fixed this.
```

이 한 줄에서 LLM이 얻는 정보: 정확한 시작/종료 시각, 화자, 필러 유무, 직전 구와의 침묵 길이.

### Layer 2 — 시각 확인 (필요할 때만)

`timeline_view.py`가 지정 구간의 **필름스트립 + 음파 리본 + 단어 라벨 + 침묵 구간 음영** PNG를 합성합니다.
애매한 멈춤, 재촬영 비교, 컷 지점 검증 등 **결정 지점에서만** 호출합니다 (스캔 루프 금지).

> browser-use가 LLM에게 스크린샷 대신 구조화된 DOM을 주는 것과 같은 발상을 영상에 적용한 것입니다.

---

## 4. 파이프라인

```
전사 ──> 압축 ──> LLM 추론 ──> EDL ──> 렌더 ──> 자체검수
                                                   │
                                      문제? 수정 + 재렌더 (최대 3회)
```

### EDL (Edit Decision List) 형식

```json
{
  "version": 1,
  "sources": {"C0103": "/abs/path/C0103.MP4"},
  "ranges": [
    {"source": "C0103", "start": 2.42, "end": 6.85, "beat": "HOOK",
     "quote": "...", "reason": "가장 깔끔한 딜리버리. 38.46초 말실수 전에 끊음."}
  ],
  "grade": "warm_cinematic",
  "overlays": [
    {"file": "edit/animations/slot_1/render.mp4", "start_in_output": 0.0, "duration": 5.0}
  ],
  "subtitles": "edit/master.srt",
  "total_duration_s": 87.4
}
```

- `beat` — 장면의 서사적 역할 (HOOK / PROBLEM / SOLUTION / BENEFIT / CTA 등)
- `reason` — **AI가 그렇게 결정한 근거**. 납품용 설명 자료로 그대로 쓸 수 있는 차별 포인트

### 자체검수 (Self-Eval)

렌더링 완료 후 AI가 **자기 산출물**에 `timeline_view`를 돌려 모든 컷 경계(±1.5초)를 점검합니다.

1. 컷 지점의 시각적 불연속 / 플래시 / 점프
2. 경계의 음파 스파이크 (30ms 페이드를 빠져나간 오디오 팝)
3. 자막이 오버레이에 가려짐 (하드룰 1 위반)
4. 오버레이 정렬 오류 / 엉뚱한 프레임 노출 (하드룰 4 위반)

추가로 처음 2초, 마지막 2초, 중간 2~3지점을 샘플링해 색감 일관성과 자막 가독성을 확인하고,
`ffprobe`로 출력 길이가 EDL 기대값과 맞는지 검증합니다.
문제가 있으면 수정 → 재렌더 → 재검수. **최대 3회**로 상한을 두고, 그 후엔 사용자에게 보고합니다.

### 산출물 디렉터리

```
<영상폴더>/
├── <원본 파일 — 손대지 않음>
└── edit/
    ├── project.md              세션 메모리 (매 세션 append)
    ├── takes_packed.md         LLM의 주 열람 뷰
    ├── edl.json                편집 결정
    ├── transcripts/<name>.json 캐시된 Scribe 원본
    ├── animations/slot_<id>/   애니메이션 소스 + 렌더 + 근거
    ├── clips_graded/           색보정 + 페이드 적용된 구간 추출물
    ├── master.srt              출력 타임라인 기준 자막
    ├── downloads/              yt-dlp 산출물
    ├── verify/                 검수용 프레임 / 타임라인 PNG
    ├── preview.mp4
    └── final.mp4
```

---

## 5. 12개 하드 룰 (프로덕션 정확성 — 타협 불가)

어기면 **조용히 실패(silent failure)** 하는 항목들입니다. ffmpeg 실무 노하우의 결정판입니다.

| # | 규칙 | 안 지키면 |
|---|---|---|
| 1 | 자막은 필터 체인 **맨 마지막**에 적용 | 오버레이가 자막을 덮어 자막이 사라짐 |
| 2 | 구간별 추출 → `-c copy` 무손실 병합 | 오버레이 추가 시 구간마다 이중 인코딩 |
| 3 | 모든 구간 경계에 30ms 오디오 페이드 | 컷마다 가청 팝("뻑") 발생 |
| 4 | 오버레이는 `setpts=PTS-STARTPTS+T/TB` | 애니메이션 중간 프레임부터 노출 |
| 5 | 마스터 SRT는 출력 타임라인 오프셋 기준 | 구간 병합 후 자막 싱크 전면 붕괴 |
| 6 | 단어 중간에서 절대 컷 금지 | 단어가 잘린 부자연스러운 컷 |
| 7 | 컷 경계에 30~200ms 패딩 | Scribe 타임스탬프의 50~100ms 드리프트 노출 |
| 8 | 단어 단위 verbatim ASR만 사용 | SRT/구 단위는 1초 미만 침묵 정보 손실 |
| 9 | 전사 결과를 소스별 캐싱 | 유료 API 비용 중복 발생 |
| 10 | 애니메이션 다수는 병렬 서브에이전트 | 순차 실행 시 소요 시간 N배 |
| 11 | 전략 승인 전 컷 금지 | 의도와 다른 결과물에 렌더 시간 낭비 |
| 12 | 모든 산출물은 `<영상폴더>/edit/` | 프로젝트 디렉터리 오염 |

**관련 공식 수식**

- 오디오 페이드: `afade=t=in:st=0:d=0.03,afade=t=out:st={dur-0.03}:d=0.03`
- 자막 오프셋: `output_time = word.start - segment_start + segment_offset`

---

## 6. helpers/ 상세

### transcribe.py (241줄)

- ffmpeg로 16kHz 모노 PCM WAV 추출 → Scribe 업로드 → `transcripts/<stem>.json` 저장
- 캐시 히트 시 업로드를 건너뜀 (하드룰 9)
- `count_audio_tracks()` — 다중 오디오 트랙 감지. OBS는 트랙0=게임, 트랙1=마이크 → `--audio-track`
- `peak_dbfs()` — 업로드 전 피크 측정. **-60dBFS 미만(무음)이면 업로드 거부**하여 비용 낭비 방지
- 트랙 번호가 전사 파일명에 포함되어(`<stem>.track1.json`) 트랙별 전사가 섞이지 않음

### transcribe_batch.py (126줄)

- `ThreadPoolExecutor`로 기본 4워커 병렬 전사. 캐시된 파일은 자동 스킵
- `transcript_path()`를 단일 소스로 공유하여 캐시 판정이 어긋나지 않음

### pack_transcripts.py (206줄)

- Scribe의 `word` / `spacing` / `audio_event` 엔트리를 구(phrase) 단위로 묶음
- 분절 기준: **침묵 0.5초 이상 OR 화자 변경**
- 결과물은 원본 JSON의 약 1/10 토큰으로 단어 단위 정밀도 유지
- UTF-8로 출력 (cp1252 인코딩 오류 수정 이력 있음)

### render.py (771줄) — 가장 큰 파일

- **HDR → SDR 톤매핑**: `color_transfer`로 HLG(`arib-std-b67`) / PQ(`smpte2084`) 감지 →
  `zscale=t=linear:npl=100 → format=gbrpf32le → zscale=p=bt709 → tonemap=hable:desat=0 → zscale=t=bt709:m=bt709:r=tv → yuv420p`
  (아이폰 기본 HLG 영상을 그냥 변환하면 색이 터지는 문제 해결)
- **세로영상 처리**: 회전 메타데이터까지 반영해 감지, 세로는 높이 기준 스케일
- **프레임레이트 보존**: 소스 fps를 probe해 유지, `--fps`로 강제 가능 (24 강제 문제 수정 이력)
- **품질 3단계**: `--draft` 720p/ultrafast/CRF28 → `--preview` 1080p/medium/CRF22 → 기본 1080p/fast/CRF20
- **라우드니스 정규화 2패스**: `-14 LUFS / -1 dBTP / LRA 11` — 유튜브·틱톡·인스타 표준
- **자막 자동 생성**: 전사 + EDL 오프셋 → `master.srt`, 2단어 UPPERCASE 청크
- 자막 스타일 `SUB_FORCE_STYLE`: Helvetica 18 Bold, 흰색 + 검은 외곽선, `Alignment=2`, **`MarginV=90`**
  (libass가 `PlayResY=288` 기준으로 스케일하므로, 모든 종횡비에서 하단 약 30% 위에 배치 →
  틱톡/릴스/쇼츠 UI 안전영역 회피. 특별한 이유 없이 75 미만으로 내리지 말 것)

### grade.py (375줄)

- 프리셋: `subtle`, `neutral_punch`, `warm_cinematic`, `none`
- **자동 모드(기본)**: ffmpeg `signalstats`로 프레임 N장 샘플 → 평균 밝기 / RMS 대비 / 채도 계산 →
  클립별 보정 필터 생성. **모든 축 ±8% 상한.** 크리에이티브 LUT·틸/오렌지 스플릿은 적용하지 않음
- 멘탈 모델은 ASC CDL: `out = (in * slope + offset) ** power` → slope=하이라이트, offset=섀도우, power=미드톤
- 하드 룰: 색보정은 **구간 추출 시점에 적용**(병합 후 적용은 이중 인코딩). 스킨톤 테스트 없이 과한 보정 금지

### timeline_view.py (392줄)

- 구간에서 프레임 N장 추출 → 가로 필름스트립 합성
- librosa로 음파 엔벨로프 계산 → 리본 그래프
- 전사에서 단어 라벨 오버레이 + 0.4초 이상 침묵 구간 음영 표시
- 코드 주석에 "스캔 루프로 돌리지 말 것, 온디맨드 드릴다운 도구" 명시

---

## 7. 설치 및 사용법

### 방법 A — 설치 프롬프트 복붙 (권장)

Claude Code / Codex / Hermes 등 셸 접근 가능한 에이전트에 아래를 붙여넣습니다.

```text
Set up https://github.com/bmshin94/video-use for me.

Read install.md first to install this repo, wire up ffmpeg, register the skill
with whichever agent you're running under, and set up the ElevenLabs API key —
ask me to paste it when you need it. Then read SKILL.md for daily usage,
and always read helpers/ because that's where the editing scripts live.
After install, don't transcribe anything on your own — just tell me it's
ready and wait for me to drop footage into a folder.
```

`install.md`는 **사람이 아니라 AI가 읽는 설치 지시서**입니다 (self-installing repo 패턴).
에이전트가 clone → 의존성 설치 → ffmpeg 확인 → 스킬 등록 → API 키 요청까지 수행합니다.

### 방법 B — 수동 설치

```bash
# 1. 안정적인 경로에 클론
git clone https://github.com/bmshin94/video-use ~/Developer/video-use
cd ~/Developer/video-use

# 2. 에이전트 스킬 디렉터리에 심볼릭 링크 (디렉터리 전체!)
mkdir -p ~/.claude/skills
ln -sfn ~/Developer/video-use ~/.claude/skills/video-use          # Claude Code
# ln -sfn ~/Developer/video-use ~/.codex/skills/video-use          # Codex

# 3. 파이썬 의존성
uv sync                 # 또는: pip install -e .

# 4. ffmpeg (필수) / yt-dlp (선택)
brew install ffmpeg                                    # macOS
# sudo apt-get update && sudo apt-get install -y ffmpeg  # Debian/Ubuntu
# sudo pacman -S ffmpeg yt-dlp                           # Arch
brew install yt-dlp

# 5. API 키
cp .env.example .env
printf 'ELEVENLABS_API_KEY=%s\n' "$KEY" > .env
chmod 600 .env

# 6. 검증
python helpers/timeline_view.py --help >/dev/null && echo "helpers OK"
ffprobe -version | head -1
```

> **주의**: 심볼릭 링크는 반드시 **디렉터리 전체**를 걸어야 합니다.
> `SKILL.md`가 헬퍼를 `transcribe.py` 같은 짧은 이름으로 참조하므로 `helpers/`가 `SKILL.md` 옆에 있어야 합니다.

> **Windows**: 이 저장소는 macOS 기준으로 작성되었습니다. WSL2(Ubuntu) 사용을 권장합니다.
> 네이티브로 쓰려면 ffmpeg를 PATH에 등록하고, 폰트 경로(`/System/Library/Fonts/Menlo.ttc`)를 교체해야 합니다.

### 일상 사용

```bash
cd /path/to/your/videos
claude                  # 또는 codex, hermes 등
```

세션에서 자연어로 지시합니다.

- `edit these into a launch video`
- `이거 60초 세로 숏폼으로, 자막 넣어서 만들어줘`
- `테이크 목록 뽑고 전략 제안해줘`
- `3번째 컷 좀 더 길게` (피드백도 자연어)

### 헬퍼 직접 실행 (AI 없이도 동작)

```bash
python helpers/transcribe.py video.mp4 --num-speakers 2
python helpers/transcribe.py video.mp4 --audio-track 1          # OBS 마이크 트랙
python helpers/transcribe_batch.py ~/videos --workers 4
python helpers/pack_transcripts.py --edit-dir ~/videos/edit
python helpers/timeline_view.py video.mp4 42.0 48.0 -o check.png
python helpers/grade.py in.mp4 -o out.mp4 --preset warm_cinematic
python helpers/grade.py --analyze in.mp4
python helpers/grade.py --list-presets
python helpers/render.py edl.json -o final.mp4 --build-subtitles
python helpers/render.py edl.json -o preview.mp4 --preview
python helpers/render.py edl.json -o draft.mp4 --draft
python helpers/render.py edl.json -o out.mp4 --fps 30
python helpers/render.py edl.json -o out.mp4 --no-loudnorm
```

### 업데이트

```bash
cd ~/Developer/video-use && git pull --ff-only
uv sync    # 의존성이 변경된 경우
```

심볼릭 링크 방식이므로 다음 실행 시 자동 반영됩니다.

---

## 8. 정체성 — 플러그인? 스킬? MCP?

### 결론: **스킬(Agent Skill)**

근거는 `SKILL.md` 상단의 YAML frontmatter + 파일명 규약 + `~/.claude/skills/` 설치 방식입니다.

```yaml
---
name: video-use
description: Edit any video by conversation. Transcribe, cut, color grade, ...
---
```

### 세 가지 개념 비교

| | 스킬 (Skill) | MCP | 플러그인 (Plugin) |
|---|---|---|---|
| 정체 | AI가 읽는 지침서 + 스크립트 | 서버 프로토콜 | 패키지 번들 |
| 형태 | `SKILL.md` + 동반 파일 | 상주 서버 프로세스 | `.claude-plugin/` + marketplace |
| 통신 | 없음 (파일 읽기) | JSON-RPC (stdio/HTTP) | 없음 |
| 설치 | 디렉터리 복사 / 심링크 | 설정에 서버 등록 | `/plugin install` |
| 구성 | 문서 + 스크립트 | tools / resources / prompts | 스킬+커맨드+훅+에이전트+MCP |
| 비유 | 업무 매뉴얼 | 전동 공구 | 공구함 세트 |
| video-use | **해당** | 아님 | 아님 |

### MCP가 아닌 이유

- `mcp.json` 등 서버 설정 파일 없음
- MCP SDK 의존성 없음 (`pyproject.toml`에 `requests, librosa, matplotlib, pillow, numpy`만)
- `stdio` / `tools/list` / `tools/call` 핸들러 없음
- 상주 프로세스 없음 — AI가 `Bash`로 파이썬 스크립트를 직접 실행

### 중첩 스킬 구조

```
video-use/SKILL.md                 항상 로드 (23KB)
└── skills/manim-video/SKILL.md    Manim 작업 시에만 로드
    └── references/ (14개 문서)     해당 세부 작업 시에만 로드
```

컨텍스트를 계층적으로 지연 로딩하는 패턴입니다.

---

## 9. API 토큰 요구사항

### 필수: ElevenLabs API 키 1개

```bash
# <저장소 루트>/.env
ELEVENLABS_API_KEY=...
```

키 탐색 순서 (`helpers/transcribe.py`):

1. `<저장소 루트>/.env`
2. 현재 디렉터리 `.env`
3. 환경변수 `ELEVENLABS_API_KEY`
4. 모두 없으면 종료

사용처는 Scribe 전사 API 하나입니다.

```python
SCRIBE_URL = "https://api.elevenlabs.io/v1/speech-to-text"
data = {
    "model_id": "scribe_v1",
    "diarize": "true",
    "tag_audio_events": "true",
    "timestamps_granularity": "word",
}
```

### 필요하지 않은 것

| 서비스 | 필요 | 비고 |
|---|---|---|
| OpenAI / Anthropic API | 불필요 | Claude Code 자체가 AI 역할 |
| ElevenLabs TTS | 불필요 | STT(전사)만 사용 |
| Google Cloud / AWS | 불필요 | 사용하지 않음 |
| Remotion 라이선스 | 불필요 | 개인/소규모 무료 |
| HyperFrames / Manim / ffmpeg / yt-dlp | 불필요 | 오픈소스 |

### 비용 절약 장치 3개

1. **캐싱** — `transcripts/*.json` 존재 시 API 호출 생략
2. **무음 감지** — 피크 -60dBFS 미만이면 업로드 거부 (잘못된 트랙으로 비용 낭비 방지)
3. **설치 중 전사 금지** — `install.md`에 명시 ("Scribe costs real money")

### 보안

- `.gitignore`에 `.env`, `.env.local`, `*.pem`, `*.key` 등록
- `chmod 600 .env` 권한 잠금
- "키를 툴 출력으로 에코하지 말 것" 지침
- `.env`는 저장소 루트에만 작성, 사용자 영상 폴더에는 쓰지 않음
- 키 검증은 무료 엔드포인트 사용: `GET https://api.elevenlabs.io/v1/user` (200=정상, 401=잘못된 키)

### 대안 ASR 검토

| 대안 | 장점 | 한계 |
|---|---|---|
| Whisper 로컬 | 무료, 오프라인 | 필러를 정규화해 지움 → 편집 신호 상실 (안티패턴에 명시) |
| Deepgram Nova | 단어 타임스탬프 + 화자분리 | 오디오 이벤트 태그 없음 |
| AssemblyAI | 기능 풍부 | 오디오 이벤트 지원 제한 |
| WhisperX | 단어 정렬 추가 | 설정 복잡, 필러 문제 동일 |

핵심은 **오디오 이벤트 + 단어별 타임스탬프 + 화자분리 + verbatim(필러 보존)** 네 가지를
한 번에 제공하는 점이고, 그래서 Scribe가 선택되었습니다.

---

## 10. AI 에이전트 설계 관점의 가치 (패턴 10가지)

영상 편집보다 이쪽이 더 큰 가치일 수 있습니다. 재사용 가능한 설계 패턴 모음입니다.

### 패턴 1. 컨텍스트 압축 (Context Compression)

원본이 컨텍스트 창보다 1000배 클 때, 의사결정에 필요한 신호만 남긴 **대리 표현**을 만든다.
압축했으나 판단 능력은 유지 — 단어 타임스탬프(정밀 컷), 침묵 길이(안전한 컷 지점),
화자 ID(핸드오프), 오디오 이벤트(명장면 보존).

### 패턴 2. 텍스트 우선 + 온디맨드 비주얼 (Layered Context)

싼 신호(텍스트)는 항상 로드, 비싼 신호(이미지)는 결정 지점에서만. 멀티모달 비용 폭발 회피.

### 패턴 3. 하드 룰 vs 예술적 자유 분리

- 하드 룰 12개: 어기면 조용히 실패 → 절대 준수
- 그 외 모든 수치/프리셋/폰트/색상: "worked example"일 뿐, 재량

전부 강제하면 경직되고, 전부 자유면 깨진 결과가 나옵니다. 분리가 핵심입니다.
SKILL.md 원칙 6: *"Invent freely. ... Do not wait for permission."*

### 패턴 4. 자체 검증 루프 + 반복 상한

AI가 자기 산출물을 검사. 검사 항목이 **구체적**이어야 작동합니다
("잘 됐는지 확인해" ❌ / "이 4가지를 이 방법으로 확인해" ✅).
무한 루프 방지를 위해 최대 3회 상한 후 사용자에게 보고.

### 패턴 5. 전략 승인 게이트 (Human-in-the-Loop)

비용이 큰 작업 **직전에만** 사람을 끼웁니다. 모든 단계마다가 아닙니다.
질문은 고정 체크리스트가 아니라 **소재에 맞춰 매번 다르게** 구성합니다.

### 패턴 6. 병렬 서브에이전트

애니메이션 N개 = 서브에이전트 N개 동시 실행. 총 소요 시간 ≈ 가장 느린 하나.
서브에이전트 프롬프트 10개 필수 항목:

1. 한 문장 목표 ("Build ONE animation: [spec]. Nothing else.")
2. 절대 출력 경로
3. 정확한 기술 스펙 (해상도, fps, 코덱, pix_fmt, CRF, duration)
4. 스타일 팔레트 (구체적 RGB/hex 값)
5. 폰트 경로 + 인덱스
6. 프레임별 타임라인 (이징 포함)
7. 안티 리스트 ("no chrome, no extras, no titles unless specified")
8. 코드 패턴 참조 (인라인 복사, 슬롯 간 import 금지)
9. 산출물 체크리스트 (ffprobe로 길이 검증까지)
10. **"Do not ask questions. If anything is ambiguous, pick the most obvious interpretation and proceed."**

10번이 중요한 이유: 서브에이전트는 부모 컨텍스트가 없어 질문하면 교착됩니다.
또한 "1 서브에이전트 = 1 파일" 규칙으로 파일명 충돌을 막습니다.

### 패턴 7. 세션 메모리 영속화

```markdown
## Session N — YYYY-MM-DD
**Strategy:** 접근 방식 한 문단
**Decisions:** 테이크 선택, 컷, 색보정, 애니메이션 + 이유
**Reasoning log:** 비자명한 결정의 한 줄 근거
**Outstanding:** 미룬 항목
```

시작 시 읽고 한 문장으로 요약 후 "이어서 할까요?" 질문. 장기 프로젝트 에이전트의 필수 패턴.

### 패턴 8. 자기 설치형 레포 (Self-Installing Repo)

`install.md`가 AI를 향해 작성됨. 실패 모드까지 문서화
("sudo가 필요하면 정확한 명령을 알려주고 대기할 것. 비밀번호를 만들어내지 말 것").

### 패턴 9. 안티패턴 명시

"하지 말 것"을 적는 것이 긍정 지시보다 효과적일 때가 많습니다.

- 계층적 사전 계산 포맷 → 과설계
- 수동 튜닝 점수 함수 → *"The LLM picks better than any heuristic you'll write"*
- Whisper SRT / 구 단위 출력 → 침묵 데이터 손실
- CPU 로컬 Whisper → 느리고 필러 정규화
- 오버레이 전 자막 번인 → 자막 가려짐
- 오버레이 있는데 단일 패스 필터그래프 → 이중 인코딩
- linear 이징 → 로봇 같음
- 순차 서브에이전트 → 항상 병렬
- 전략 확인 전 편집 → 금지
- 영상 종류 가정 → 보고, 묻고, 편집

### 패턴 10. 중첩 스킬 + 지연 로딩

필요할 때만 하위 스킬/참조 문서를 읽어 컨텍스트를 절약.

### 설계 철학 7계명 (SKILL.md Principle)

1. LLM은 원본 전사 + 온디맨드 비주얼로 추론한다
2. 오디오가 1차, 비주얼은 뒤따른다
3. 묻고 → 확인하고 → 실행하고 → 반복하고 → 저장한다
4. 일반화하라 (종류를 가정하지 말고 보고 물어라)
5. 예술적 자유가 기본값 (모든 구체 값은 예시)
6. 자유롭게 발명하라 (허락을 기다리지 말고)
7. 사용자에게 보여주기 전에 스스로 검증하라

### 다른 도메인 적용

| 무거운 원본 | 가벼운 텍스트 변환 | 사례 |
|---|---|---|
| 영상 30,000프레임 | 전사 12KB | video-use |
| 웹페이지 스크린샷 | DOM 구조 텍스트 | browser-use |
| PDF 500페이지 | 목차 + 섹션 요약 | 문서 분석 AI |
| 음악 10분 | 비트맵 + 코드 진행 | 음악 분석 AI |
| 코드베이스 100만줄 | 심볼 맵 + import 그래프 | 코딩 에이전트 |
| 엑셀 100만행 | 스키마 + 통계 요약 | 데이터 분석 AI |
| CCTV 24시간 | 이벤트 로그 텍스트 | 보안 감시 AI |

---

## 11. 애니메이션 엔진 4종

| 엔진 | 적합한 경우 |
|---|---|
| **HyperFrames** | HTML/CSS/GSAP — 제품 UI 모션, 웹→영상 캡처, 키네틱 타이포, 투명 WebM 오버레이 |
| **Remotion** | React 컴포넌트 상태/재사용 프리미티브, 기존 Remotion 브랜드 시스템 |
| **Manim** | 수식 유도, 상태 머신, 그래프 모프, 형식적 다이어그램 |
| **PIL + PNG 시퀀스 + ffmpeg** | 단순 오버레이 카드, 카운터, 타이프라이터, 프로그레스 (원본 런칭 영상에 사용) |

### 애니메이션 제작 규칙

- **이징은 절대 linear 금지**: 단일 등장은 `ease_out_cubic`, 연속 드로잉은 `ease_in_out_cubic`

```python
def ease_out_cubic(t):    return 1 - (1 - t) ** 3
def ease_in_out_cubic(t):
    if t < 0.5: return 4 * t ** 3
    return 1 - (-2 * t + 2) ** 3 / 2
```

- **타이핑 텍스트 앵커**: 부분 문자열이 아니라 **전체 문자열 폭** 기준 중앙정렬 (안 하면 텍스트가 왼쪽으로 밀림)
- **페이오프 싱크**: 핵심 단어 발화 시점에 완성 프레임이 오도록 `reveal_duration`만큼 미리 시작
- **길이 기준**: 내레이션 동기 설명은 하한 3초, 단순 카드 5~7초, 복잡한 다이어그램 8~14초 /
  비트 동기 액센트는 0.5~2초 / 마지막 프레임 1초 이상 유지 / 보이스오버 위에서는 `내레이션 길이 + 1초` 이상
- **독립 요소 동시 노출 금지** — 시선은 두 개의 새 정보를 동시에 추적할 수 없음

---

## 12. React / PHP 이식 가능성

### 결론: **완전 포팅은 비추, "래핑"이 정답**

### 구성요소별 이식 난이도

| 구성요소 | 현재 | React | PHP | 비고 |
|---|---|---|---|---|
| ElevenLabs API 호출 | requests | 가능 (fetch) | 가능 (cURL) | 단순 HTTP |
| ffmpeg 호출 | subprocess | Node만 가능 | 가능 (exec) | 브라우저 React는 불가 |
| 전사 → 구 단위 압축 | 순수 로직 | 쉬움 | 쉬움 | 배열 처리 |
| 음파 분석 | **librosa** | 대체 필요 | 대체 필요 | **최대 난관 1** |
| 이미지 합성 | PIL | canvas/sharp | GD/Imagick | 가능하나 노동 |
| 그래프 렌더링 | matplotlib | 직접 구현 | 직접 구현 | 가능하나 노동 |
| 색보정 통계 | ffmpeg signalstats | 파싱만 | 파싱만 | ffmpeg가 처리 |
| **AI 판단** | Claude Code | Anthropic API | Anthropic API | **최대 난관 2 (비용 구조 변경)** |

AI 비용 구조가 바뀌는 점이 가장 중요합니다.
현재는 Claude Code 구독으로 동작하지만, 자체 앱은 Anthropic API 종량제가 됩니다.
(사용자가 자기 키를 넣게 하면 해결되나 진입장벽이 생깁니다.)

### 권장 아키텍처 — 파이썬은 유지, 껍데기만 추가

```
React 프론트엔드 (Next.js / Vite)
  - 영상 드래그&드롭 업로드
  - 전사 결과 타임라인 뷰어 (단어 클릭 → 해당 시점 점프)
  - 컷 포인트 시각 편집 (드래그 조절)
  - EDL 미리보기 / 수정
  - 렌더링 진행률 (WebSocket)
  - 결과 재생 + 다운로드
        │  REST / WebSocket
FastAPI 백엔드 (Python)
  POST /api/transcribe  → transcribe_batch.py   (그대로 재사용)
  GET  /api/takes       → pack_transcripts.py   (그대로 재사용)
  POST /api/timeline    → timeline_view.py      (그대로 재사용)
  POST /api/render      → render.py             (그대로 재사용)
  POST /api/strategy    → Anthropic API (전략 제안 / EDL 생성)
  WS   /api/progress    → 진행률 스트리밍
  + Celery / RQ 작업 큐 (렌더링은 장시간 작업)
```

이 방식의 장점: `helpers/` 2,111줄을 한 줄도 버리지 않고, librosa/PIL/matplotlib 대체 고민이 없으며,
업스트림 업데이트를 `git pull`로 계속 받을 수 있습니다.

### PHP(라라벨) 예시

```php
class VideoUseService
{
    private string $helpers;

    public function __construct()
    {
        $this->helpers = config('videouse.path') . '/helpers';
    }

    public function transcribeBatch(string $videosDir): array
    {
        $cmd = sprintf(
            'python3 %s/transcribe_batch.py %s --workers 4 2>&1',
            escapeshellarg($this->helpers),   // escapeshellarg 필수
            escapeshellarg($videosDir)
        );
        exec($cmd, $output, $code);
        if ($code !== 0) {
            throw new \RuntimeException('transcribe failed: ' . implode("\n", $output));
        }
        return $output;
    }
}
```

렌더링은 수 분이 걸리므로 라라벨 큐 잡(`ShouldQueue`, `$timeout = 3600`)으로 처리합니다.

### 완전 포팅이 비추인 이유

- librosa 대체 부재 (음파 엔벨로프를 FFT로 직접 구현해야 함)
- HDR 톤매핑 / 회전 메타데이터 / 2패스 라우드니스 등 ffmpeg 노하우 2,111줄을 재디버깅
- 업스트림 버그 수정 반영이 수동이 됨
- `ffmpeg.wasm`은 5~10배 느리고 메모리 한계로 4K 처리 곤란
- AI 비용 구조가 구독제 → 종량제로 변경

### React가 강점인 영역

- 타임라인 에디터 (드래그 컷 조절, 파형, 썸네일 스크러빙)
- 전사 뷰어 (단어 클릭 점프, 필러 하이라이트)
- EDL 시각화 (테이크별 색상 구분)
- **Remotion** — 애니메이션 자체를 React로 작성 (SKILL.md 공식 옵션)

### PHP가 강점인 영역

- 사내 영상 자동화 포털 (기존 라라벨 시스템 연동)
- 결제/구독 관리 레이어
- 작업 완료 알림 시스템

### 단계별 로드맵

```
1단계 (1~2주)  React + FastAPI 로컬 웹 UI. AI 없이 수동 모드로 작동 확인
2단계 (2~3주)  Anthropic API로 전략 제안 + EDL 자동 생성 (SKILL.md → 시스템 프롬프트)
3단계 (3~4주)  Celery/RQ + Redis 작업 큐, WebSocket 진행률
4단계 (4~6주)  멀티유저 인증, 플랜, 결제, S3/R2 스토리지, 사용량 미터링
5단계          GPU 렌더 노드(NVENC), CDN, 워커 오토스케일
```

---

## 13. 유튜브 강의 시리즈 기획

### 소재로서의 강점

| 요소 | 이유 |
|---|---|
| 반전 있는 훅 | "AI가 영상을 안 보고 편집한다" |
| 충격적인 숫자 | "4,500만 토큰 → 12KB, 2,700배" |
| 시각적 매력 | 필름스트립 + 음파 그래프 |
| 메타 구성 | **이 도구로 이 강의 영상을 편집** → 설명이 곧 증거 |
| 실용성 | 영상 편집은 보편적 수요 |
| 깊이 확장 | AI 설계 패턴으로 고급 시청자 유입 |
| 접근성 | MIT 오픈소스, 즉시 따라하기 가능 |
| 블루오션 | 한국어 콘텐츠 거의 없음 |

### 8편 시리즈 구성

| EP | 제목 | 길이 | 핵심 |
|---|---|---|---|
| 01 | AI가 영상을 '보지 않고' 편집하는 법 | 10~12분 | 압축 개념 + "이 영상도 이 도구로 편집" 반전 |
| 02 | 0부터 설치: 복붙 한 줄로 끝 | 8~10분 | self-installing repo 개념, 실시간 설치 녹화 |
| 03 | 실전: 날영상 10개 → 60초 숏폼 | 15~20분 | **조회수 캐리 편**. 과정 전체를 날것으로 |
| 04 | 영상 편집 12계명 | 12~15분 | 하드룰 위반 결과를 실물 데모 (팝 소리, 사라진 자막) |
| 05 | AI 에이전트 설계의 교과서 | 15~18분 | 패턴 10개. 개발자 충성도 높음, 강의 연결 |
| 06 | 애니메이션 오버레이: 4가지 엔진 비교 | 12~15분 | 병렬 서브에이전트 시간 측정, 이징 비교 |
| 07 | 한국어 영상으로도 되나? 실험 | 12~15분 | 한국어 전사 품질, 자막 청킹, 실제 PR |
| 08 | React + FastAPI로 웹 UI 만들기 | 20~25분 | 풀스택 수요, 유료 강의 확장 최적 |

### EP.01 구성 예시

```
0:00   훅 — 숫자 충격
0:45   문제 정의 — AI는 왜 영상을 못 보나
2:00   발상 전환 — "보지 말고 읽자"
4:00   Layer 1 실물 — takes_packed.md 공개
6:30   Layer 2 — timeline_view PNG 공개
8:30   browser-use와의 평행 (DOM vs 스크린샷)
10:00  "이 영상도 이 도구로 편집했습니다"
11:00  다음 편 예고 + 구독
```

### 쇼츠 아이디어

1. 오디오 페이드 없을 때 팝 소리 (Before/After 청취) — 바이럴 가능성 높음
2. "4,500만 토큰 → 12KB" 숫자 애니메이션
3. 자막 순서 하나 틀려서 자막이 사라진 영상
4. AI가 "음..." 47개를 1초에 지우는 장면
5. 복붙 한 줄로 AI가 혼자 설치하는 타임랩스
6. linear 이징 vs cubic 이징 나란히 비교
7. 자막이 틱톡 UI에 가려지는 이유 (MarginV 설명)

### 제작 팁

- 길이 10~15분, 첫 15초에 숫자 또는 결과물 먼저
- 터미널 폰트 16pt 이상, 다크 테마, 하이라이트 줌
- GitHub 저장소 + 설치 프롬프트를 설명란 고정
- 렌더링 구간은 타임랩스/배속 처리
- **`.env` 화면 노출 절대 금지** (블러 처리)
- "2026년 10월 기준" 명시 + 핀 댓글로 업데이트 관리
- MIT 라이선스 고지 및 원작자(browser-use) 크레딧 명시

---

## 14. 수익화 전략 (10가지)

### 전체 비교

| # | 아이디어 | 난이도 | 초기비용 | 수익화 속도 | 잠재력 |
|---|---|---|---|---|---|
| 1 | 영상편집 대행 서비스 | ★☆☆☆☆ | 거의 0 | 즉시 | 중 |
| 2 | 유튜브 + 유료 강의 | ★★☆☆☆ | 거의 0 | 2~3개월 | 상 |
| 3 | 니치 스킬 팩 판매 | ★★☆☆☆ | 낮음 | 1개월 | 중 |
| 4 | 한국어 특화 포크 | ★★★☆☆ | 낮음 | 1~2개월 | 상 |
| 5 | MCP 서버 + 유료 호스팅 | ★★★☆☆ | 중간 | 2개월 | 상 |
| 6 | B2B 사내 영상 자동화 | ★★★★☆ | 중간 | 3~6개월 | 최상 |
| 7 | 숏폼 자동편집 SaaS | ★★★★☆ | 높음 | 4~6개월 | 최상 |
| 8 | 모바일/웹 앱 | ★★★★★ | 매우높음 | 6개월+ | 상 |
| 9 | 콘텐츠 공장 (직접 운영) | ★★★☆☆ | 중간 | 2~3개월 | 중 |
| 10 | 템플릿 / 프리셋 마켓 | ★★☆☆☆ | 낮음 | 1개월 | 하 |

### 1. 영상 편집 대행 서비스 — 즉시 시작 가능

개발 0줄, 리스크 거의 0, 시장 검증 동시 수행.

| 상품 | 가격대 | 포함 |
|---|---|---|
| 베이직 | 3~5만원 | 필러 제거 + 컷 정리 + 기본 색보정 (~5분) |
| 스탠다드 | 8~15만원 | + 자막 번인 + 숏폼 세로 1개 |
| 프리미엄 | 20~40만원 | + 애니메이션 2~3개 + 숏폼 3개 |
| 월정액 | 50~150만원 | 월 영상 N개 (유튜버/기업) — MRR 확보 |

**타겟 순서**: 1인 유튜버/강사 → 숏폼 크리에이터 → 스타트업 마케팅팀 → 온라인 강의 제작자 → 종교/교육 기관

**차별점**: 처리 시간이 2~3일 → 2~3시간. 게다가 EDL의 `reason` 필드를 그대로 "편집 근거 문서"로 납품.

**실행**: 1주차 포트폴리오(Before/After) → 2주차 크몽/숨고 등록 → 3주차 지인 무료 샘플로 후기 확보
→ 4주차 인스타/스레드 Before/After 쇼츠 → 2개월 가격 인상 + 월정액 → 3개월 자동화로 처리량 증대

**리스크**: 시간 노동이라 스케일 한계 → 1단계로만 활용하고 상위 아이디어로 전환

### 2. 유튜브 + 유료 강의 — 가성비 최고

```
유튜브 무료 시리즈 (유입 깔때기)
 ├─ 애드센스          월 10~100만원 (구독 1만 기준)
 ├─ 채널 멤버십        월 5~30만원 (설정파일/템플릿 제공)
 ├─ 유료 강의          5~15만원 × 수백명  ← 가장 큰 수익
 ├─ 대행 문의          아이디어 1번과 결합
 ├─ 제휴 수익          ElevenLabs 추천, 호스팅 추천
 └─ 기업 강의/컨설팅    회당 100~500만원
```

**강의 포지셔닝이 핵심**: "영상 편집 강의"는 틈새, **"AI 에이전트 설계 강의"는 시장 10배**.
video-use는 그 강의의 살아있는 교재입니다.

**커리큘럼 안**

```
Part 1. 왜 AI는 영상을 못 보는가 (컨텍스트의 경제학)
Part 2. 압축 설계: 4,500만 → 1만 토큰
Part 3. 하드룰 vs 자유: 프롬프트 설계의 이분법
Part 4. 자체검증 루프
Part 5. 병렬 서브에이전트
Part 6. 세션 메모리와 장기 에이전트
Part 7. 실습 — 내 도메인에 적용 (문서/코드/데이터)
Part 8. 실습 — React + FastAPI 웹 UI
Part 9. 배포 — 스킬/MCP 패키징
```

### 3. 니치 특화 스킬 팩 판매

| 니치 | 특화 내용 | 타겟 |
|---|---|---|
| 게임 하이라이트 | OBS 다중트랙 자동처리(`--audio-track` 활용), 킬 장면 감지 | 스트리머 |
| 설교/강연 | 긴 영상 챕터 분할, 핵심 인용구 추출, 숏폼 자동 생성 | 교회/단체 |
| 온라인 강의 | 챕터 마커, 화면공유 + 얼굴 PIP, 필러 제거 극대화 | 강사 |
| 팟캐스트 → 영상 | 파형 비주얼라이저, 화자별 자막 색상 | 팟캐스터 |
| 면접/인터뷰 | 질문/답변 자동 구분, 질문별 챕터 | HR / 미디어 |
| 운동/댄스 | 비트 싱크 컷, 반복 구간 감지 | 피트니스 |
| 요리 | 단계별 챕터, 재료 오버레이 | 푸드 크리에이터 |
| 부동산 | 공간별 컷, 매물정보 오버레이 | 중개사 |

가격: 단품 3~10만원 / 번들 3종 15~20만원 / 업데이트 구독 월 1~3만원

구성: `SKILL.md`(니치 룰) + `presets/` + `templates/` + `prompts/` + `examples/`

### 4. 한국어 특화 포크 — 블루오션

현재 영어 기준 설계의 한국어 한계:

- 자막 청킹이 2단어 UPPERCASE → 한국어에 부적합
- 필러 사전이 `umm`, `uh` → 한국어는 "음", "어", "그니까", "뭐지", "이제"
- 폰트가 Helvetica / Menlo → 한글 미지원
- 한국어는 어절·어미 경계 컷이 자연스러움

개선 방향:

```python
# 1. 한국어 자막 청킹 — 어절 단위 + 글자수 기준
def chunk_korean(words, max_chars=12): ...

# 2. 한국어 필러 사전
KO_FILLERS = {"음", "어", "아", "그", "저", "뭐", "이제",
              "그니까", "그러니까", "뭐지", "어쨌든", "약간"}

# 3. 한글 폰트 자동 선택
KO_FONTS = ["Pretendard", "NotoSansKR", "AppleSDGothicNeo", "NanumGothic"]

# 4. 어미/조사 경계 컷 규칙
```

무료 오픈소스 공개 → 커뮤니티·포트폴리오·스타 → 한국 크리에이터 유입 →
유료 서포트/컨설팅, 한국어 특화 SaaS, 한국어 전문 대행, 강의로 연결.
원본에 PR 환원 시 오픈소스 컨트리뷰터 이력 확보.

### 5. MCP 서버로 래핑 + 유료 호스팅

스킬이라 Claude Code/Codex에 한정되는 것을 MCP로 감싸면
Cursor, Windsurf, Cline, Continue, VS Code 등으로 확장됩니다.

```
video-use-mcp
├── tool: transcribe_video(path, num_speakers?, audio_track?)
├── tool: pack_transcripts(edit_dir)
├── tool: timeline_view(video, start, end) → 이미지 반환
├── tool: analyze_grade(video, start, duration)
├── tool: render_edl(edl, output, preview?, fps?)
├── resource: takes_packed.md
├── resource: project.md
└── prompt: "edit_strategy"
```

오픈코어 모델:

```
무료 (셀프 호스팅)  — 오픈소스, 본인 키, 본인 컴퓨터 렌더
유료 클라우드
  월 9,900원   전사 30분/월, 렌더 10회
  월 29,000원  전사 3시간/월, 렌더 무제한, GPU 가속
  월 99,000원  팀 계정, API 접근, 우선 큐
  → ElevenLabs 키 불필요, 서버 GPU 렌더(NVENC 5~10배)
```

MCP 생태계 초기라 선점 효과가 큽니다. 리스크는 GPU 비용과 영상 보관 책임.

### 6. B2B 사내 영상 자동화 — 수익 최대

| 업종 | 니즈 | 예상 단가 |
|---|---|---|
| 온라인 교육 | 강의 수백 개 일괄 처리, 챕터 자동 생성 | 2,000~8,000만원 |
| 미디어/방송 | 인터뷰 소재 자동 전사 + 클립 추출 | 3,000만원~ |
| 대기업 HR | 사내교육/면접 영상 아카이빙 | 2,000~5,000만원 |
| 이커머스 | 리뷰 영상 → 숏폼 대량 생산 | 1,500~4,000만원 |
| 의료/법률 | 상담 기록 전사 + 하이라이트 (보안 중요) | 3,000만원~ |
| 게임사 | 개발 영상, 트레일러 소재 관리 | 2,000만원~ |

수익 구조: 초기 구축 2,000~8,000만원 + **월 유지보수 100~500만원(안정 MRR)** + 추가 개발 + 교육/컨설팅

진입 경로: 대행으로 소형 고객 확보 → 반복 작업 발견 → 자동화 제안 → 소규모 PoC(500만원) → 본 계약 → 레퍼런스 확보

### 7. 숏폼 자동편집 SaaS — 가장 큰 그림

경쟁사: Opus Clip($19~29/월), Vizard($20~40), Klap($29~79), Descript($12~24)

차별점:

| 차별점 | 설명 |
|---|---|
| 컷 근거 설명 | EDL `reason` 필드 — 경쟁사에 없음 |
| 자연어 수정 | "3번째 컷 좀 더 길게" → 재렌더 (경쟁사는 슬라이더) |
| 한국어 최적화 | 아이디어 4번 결합 → 국내 시장 선점 |
| AI 자체 검수 | 품질 보증 |
| 애니메이션 오버레이 | 4가지 엔진 (경쟁사는 템플릿만) |
| 오픈소스 기반 | 투명성 → 개발자 신뢰 |

가격: Free 월 1개(워터마크) / Starter 14,900원 / Pro 39,900원 / Team 99,000원

스택: Next.js + FastAPI(`helpers/` 재사용) + Celery/Redis + S3·R2 + GPU 워커(NVENC)
+ Anthropic API + ElevenLabs + 토스페이먼츠/Stripe

리스크: GPU 렌더 비용(단가 계산 철저), 경쟁 치열, CAC 관리

### 8. 모바일/웹 앱

난이도 최상. 모바일 렌더링은 배터리/발열/시간 문제, 서버 렌더 필수, 앱스토어 수수료 30%,
CapCut 등 무료 강자 존재. **PWA 또는 아이디어 7번의 모바일 반응형 웹으로 대체 권장.**

### 9. 콘텐츠 공장 (직접 운영)

도구를 파는 대신 도구로 콘텐츠를 대량 생산. 처리량이 5~10배로 늘어나는 점이 핵심
(일반 편집 월 10개 → video-use 월 50~100개).

모델: 다채널 운영 / 클립 채널(긴 영상 → 숏폼) / 교육 콘텐츠 / 유튜버 수익 쉐어 파트너십
기대치: 채널 1개 월 10~50만원 × 5개 → 월 50~250만원 + 협찬
리스크: 조회수 변동성, 플랫폼 정책, AI 대량생산 콘텐츠 알고리즘 불이익, 원본 저작권

### 10. 템플릿 / 프리셋 마켓

Color Grade Pack(2~5만원) / Subtitle Style Pack(2~4만원) /
Animation Template Pack(5~15만원) / EDL Template Pack(3만원)

판매 채널: Gumroad, Lemon Squeezy, 크몽, 오투잡, 자체 랜딩
단가가 낮아 단독 수익은 제한적 — **다른 아이디어의 부가 상품으로 활용**

---

## 15. 권장 실행 순서

```
0~1개월 — 검증
  ① 대행 서비스 시작      즉시 현금 + 수요 확인 + 노하우 축적
  ② 유튜브 EP.01~03       브랜딩 + 유입 깔때기

2~3개월 — 자산 축적
  ③ 한국어 특화 포크 공개   오픈소스 신뢰도 + 포트폴리오
  ④ 유튜브 EP.04~08 + 쇼츠
  ⑤ 니치 스킬 팩 1~2개 판매

4~6개월 — 스케일
  ⑥ 유료 강의 출시         큰 수익
  ⑦ MCP 서버 공개          생태계 선점 + 재바이럴
  ⑧ B2B 리드 발굴          최대 수익

6개월+ — 선택
  ⑨ SaaS 구축 (1~8에서 수요 확인된 후에만)
  ⑩ 템플릿 마켓 (부가 상품)
```

### 순서의 근거

| 원칙 | 설명 |
|---|---|
| 현금 먼저 | 대행으로 즉시 수익 → 다음 단계 자금 조달 |
| 검증 먼저 | 수요 확인 없이 SaaS를 만드는 것이 가장 흔한 실패 |
| 자산 축적 | 유튜브/오픈소스는 시간이 지나며 복리로 성장 |
| 상호 강화 | 유튜브 → 대행 문의 → 사례 → 강의 소재 → SaaS 유입 |
| 단가 상승 | 대행(10만) → 강의(500만) → B2B(5,000만) |
| 리스크 관리 | 저위험부터, 고위험은 자금·검증 확보 후 |

### 수익 시나리오 (낙관 기준, 참고용)

```
3개월차   대행 100~300 + 유튜브 0~20 + 스킬팩 10~50      = 월 110~370만원
6개월차   대행 200~500 + 유튜브 30~100 + 강의 100~500
          + 스킬팩 30~100                                  = 월 360~1,200만원
12개월차  대행/B2B 500~3,000 + 유튜브 100~300
          + 강의 200~800 + SaaS 100~1,000                 = 월 900~5,100만원
```

핵심은 **① 대행으로 즉시 현금 + ② 유튜브로 자산 축적**을 동시에 시작하면
리스크는 거의 없고 상방은 열려 있다는 점입니다.

---

## 16. 한계와 주의사항

### 도구 자체의 한계

| 한계 | 설명 |
|---|---|
| ElevenLabs API 키 필수 | 전사 없이는 동작 불가. Scribe 크레딧 비용 발생 |
| 터미널 필수 | GUI 없음. Claude Code / Codex 등 CLI 에이전트 필요 |
| macOS 편향 | Menlo 폰트 경로, brew 중심. 리눅스/윈도우는 조정 필요 |
| 렌더링 속도 | CPU libx264. GPU 가속 미지원 |
| 음성 없는 영상 | 오디오 전사가 1차 신호라 BGM만 있는 영상은 활용 곤란 |
| 전략 승인 대기 | 완전 무인 자동화는 아님 (안전장치이기도 함) |

### 사업화 시 체크리스트

| 항목 | 주의사항 |
|---|---|
| MIT 라이선스 | LICENSE 파일 + 저작권 고지 포함 필수. 크레딧 명시 권장 |
| API 비용 | ElevenLabs 비용을 반드시 가격에 반영 (미반영 시 적자) |
| 개인정보 | 고객 영상 보관/삭제 정책 명확화, 개인정보처리방침 필수 |
| 저작권 | 고객이 영상 사용권을 보유하는지 계약서에 명시 |
| 세금 | 사업자등록, 부가세. 매출 확대 시 세무 상담 |
| 업스트림 추적 | 원본 업데이트/보안 패치 반영 |
| 커뮤니티 환원 | 개선사항은 원본에 PR → 평판 + 포트폴리오 |

### 저장소에서 발견한 개선 포인트

`SKILL.md`의 `bold-overlay` 자막 예시에는 `MarginV=35`로 기재되어 있으나,
실제 `helpers/render.py:52`의 `SUB_FORCE_STYLE`은 `MarginV=90`입니다.
커밋 `87f00c6 fix(subtitles): raise MarginV to 90 to clear vertical safe zones (#5)`에서
코드만 수정되고 문서는 갱신되지 않은 것으로 보입니다.
→ 업스트림에 보낼 첫 PR 소재로 적합합니다.

---

## 17. 버그 수정 이력 (커뮤니티 성숙도 지표)

| 커밋 | 내용 |
|---|---|
| `9575612` | transcribe: 오디오 트랙 선택, 무음 업로드 거부 (#134) |
| `b5352cf` | fix(render): 회전 메타데이터 반영한 세로영상 감지 (#137) |
| `0c40770` | fix(render): 소스 프레임레이트 보존, `--fps` 추가 (#55) |
| `1200463` | fix(render): 세로 소스 방향 보존 (#29) |
| `196d7e9` | fix(pack_transcripts): UTF-8 출력으로 cp1252 오류 회피 (#10) |
| `87f00c6` | fix(subtitles): MarginV 90으로 세로 안전영역 확보 (#5) |
| `60798b1` | fix(render): HLG/PQ 소스를 Rec.709 SDR로 톤매핑 (#6) |
| `fbcf29f` | docs: HyperFrames 애니메이션 옵션 추가 (#13) |
| `fdf749c` | MIT 라이선스 추가 (#32) |

실전 사용 중 발견된 문제들이 지속적으로 수정되어 온 저장소입니다.

---

## 18. 최종 요약

1. **정체** — Claude Code용 **Agent Skill**. MCP도, 플러그인도 아님
2. **핵심 발명** — LLM이 영상을 "보지 않고 읽게" 만드는 컨텍스트 압축 (약 2,700배)
3. **구성** — `SKILL.md`(지침) + `helpers/`(파이썬 2,111줄) + 중첩 Manim 스킬
4. **필수 토큰** — ElevenLabs API 키 1개 (Scribe 전사). 그 외 없음
5. **최대 가치** — 영상 편집 도구를 넘어 **AI 에이전트 설계 패턴 교본**
6. **이식 전략** — 완전 포팅 대신 React/PHP 프론트 + 파이썬 백엔드 래핑
7. **수익화 우선순위** — 대행(즉시) → 유튜브/강의(자산) → B2B/SaaS(스케일)
8. **콘텐츠 가능성** — 8편 유튜브 시리즈 + 쇼츠. 한국어 영역은 블루오션
9. **라이선스** — MIT. 상업적 이용 가능 (고지 의무 준수)
10. **주의** — ElevenLabs 비용, macOS 편향, CPU 렌더 속도, 음성 없는 영상 비적합
