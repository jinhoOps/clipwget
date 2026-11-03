# ClipWget 구현 계획서 (Implementation Plan)

본 문서는 [소프트웨어 설계 명세서 (software-design.md)](./software-design.md)를 바탕으로 `clipwget`을 단계별로 구현·검증하기 위한 실행 계획서입니다. 개발자 및 AI 에이전트(Claude Code, Codex, Antigravity 등) 간 협업의 기준 문서로 사용합니다.

> **동작 명세는 설계 문서가 단일 기준입니다.** 이 문서는 순서·체크리스트·검증만 다루며, 각 항목 옆 `(설계 §N)`이 해당 명세 위치입니다. 둘이 어긋나면 설계 문서를 고친 뒤 이 문서를 맞춥니다.

---

## 1. 확정된 기술 의사결정 요약 (ADR)

- **Go 모듈**: `github.com/jinhoOps/clipwget` (`CGO_ENABLED=0` 순수 Go 빌드 원칙)
- **바이너리 분리**: `clipwget.exe`(콘솔, CLI) + `clipwgetw.exe`(GUI, 트레이 데몬·프로토콜 핸들러)
- **휘발성 수집함**: 감지된 항목은 데몬 메모리에만 보관하고 앱/PC 종료 시 소멸. 디스크에는 **완료 이력만** 영속
- **공용 파이프라인**: CLI `get`과 데몬이 같은 `pipeline.Fetch`로 다운로드 (잠금·이름 확정·이력 기록 규칙을 한 곳에 둠)
- **의존성**:
  - `golang.org/x/sys` — Win32 API(클립보드, `Shell_NotifyIconW`, Named Pipe, 이름 있는 뮤텍스, `MoveFileEx`, 레지스트리)
  - `golang.org/x/net/publicsuffix` — HLS 인증 헤더 전달 판정용 eTLD+1 계산
  - `modernc.org/sqlite` — 완료 이력 (Step 1 측정 결과에 따라 JSON Lines로 대체 가능)
  - **WinRT 바인딩** — 인터랙티브 Toast 전용 (`github.com/go-ole/go-ole` 또는 `github.com/saltosystems/winrt-go`, Step 6 스파이크 후 선정)
  - 실제 버전·배포 URL·해시는 해당 Step에서 공식 자료를 확인한 뒤 고정
- **FFmpeg**: 로컬 폴더 → 시스템 `PATH` → BtbN/FFmpeg-Builds 고정 버전 ZIP을 SHA-256 검증 후 설치
- **알림**: MVP는 트레이 풍선 알림(`NIF_INFO`) → 고도화 단계에서 WinRT 인터랙티브 Toast (`Notifier` 인터페이스로 격리)
- **보관함 UI**: 온디맨드 로컬 웹 UI (`embed.FS` + `127.0.0.1:0` + OS 기본 브라우저)
- **구현 순서 원칙**: `clipwget get <url>`로 끝까지 받아지는 세로 슬라이스를 먼저 완성하고 그 위에 클립보드·트레이·알림·웹 UI를 얹음. **Step 1~5가 기본 MVP**. Step 7은 Step 5에만 의존하므로 Step 6(WinRT) 스파이크가 실패해도 진행 가능

---

## 2. 목표 디렉토리 및 패키지 구조

의존성 방향과 계층 규칙은 설계 §2.2를 따릅니다.

```text
clipwget/
├── go.mod
├── go.sum
├── docs/
│   ├── software-design.md              # 동작 명세 (단일 기준)
│   └── implementation-plan.md          # 단계별 구현 계획 및 체크리스트 (본 문서)
├── cmd/
│   ├── clipwget/
│   │   └── main.go                     # 콘솔 CLI (get / list / watch / history / setup-ffmpeg)
│   └── clipwgetw/
│       └── main.go                     # GUI (트레이 데몬 / clipwget:// 프로토콜 핸들러)
├── tools/
│   └── memspike/
│       └── main.go                     # Step 1 메모리 측정 스파이크
└── internal/
    ├── media/                          # 도메인 타입 (의존성 없음): LinkType, Source, Item, Key, 공용 오류
    │   ├── media.go
    │   ├── key.go                      # 중복 키 생성, URL 정규화, 호스트별 인증 파라미터 목록, 표시용 URL
    │   └── key_test.go
    ├── config/
    │   ├── config.go                   # 저장 경로, 자동 다운로드 토글, 포터블 모드
    │   └── config_test.go
    ├── router/
    │   ├── router.go                   # Social / HLS / Direct 판별 → media.Source
    │   ├── curl.go                     # Copy as cURL (bash / cmd) 토크나이저 (실행 금지, 헤더 개행 거부)
    │   ├── router_test.go
    │   └── curl_test.go
    ├── extractor/
    │   ├── extractor.go                # Extractor 인터페이스, 안전 HTTP 클라이언트 (사설 IP 차단, 크기·시간 제한)
    │   ├── twitter.go                  # fxtwitter / vxtwitter
    │   ├── hls.go                      # Master/Media 파서, 분리 오디오, 라이브/DRM 판정, 헤더 전달 정책
    │   ├── direct.go
    │   ├── *_test.go
    │   └── testdata/                   # API 응답 / m3u8 픽스처
    ├── naming/
    │   ├── formatter.go                # 파일명 제안 (로컬 시간, 260자, 문맥 매칭 조건)
    │   └── formatter_test.go
    ├── lock/
    │   ├── mutex_windows.go            # 이름 있는 뮤텍스 (데몬 단일 인스턴스, 미디어 키별 작업 잠금)
    │   ├── workdir.go                  # .clipwget-work/<UUID>/ 생성·소유 표식·안전 청소
    │   ├── finalize_windows.go         # MoveFileEx(덮어쓰기 금지) + (2), (3) 재시도
    │   └── *_test.go
    ├── history/
    │   ├── store.go                    # Store 인터페이스
    │   ├── sqlite.go                   # WAL, busy_timeout, upsert, user_version 마이그레이션
    │   └── sqlite_test.go
    ├── ffmpeg/
    │   ├── manager.go                  # 탐색 + 고정 버전 SHA-256 검증 설치
    │   └── manager_test.go
    ├── downloader/
    │   ├── http.go                     # 스트리밍 HTTP 다운로드, 썸네일 캐시
    │   ├── ffmpeg.go                   # -c copy 결합 (입력별 -headers, -map, -f mp4, file 제외 whitelist)
    │   └── *_test.go
    ├── pipeline/
    │   ├── pipeline.go                 # Resolve / Fetch (설계 §2.3, §7.3) — CLI와 데몬 공용
    │   └── pipeline_test.go
    ├── inbox/
    │   ├── inbox.go                    # 휘발성 메모리 수집함 (이력 모름)
    │   └── inbox_test.go
    ├── app/
    │   ├── app.go                      # Commands 구현, 워커 풀, 중복 정책 판정, 문맥 보관, 이벤트
    │   └── app_test.go
    ├── winloop/
    │   └── loop_windows.go             # UI 스레드 + 숨겨진 최상위 창 + 메시지 디스패치 (TaskbarCreated 처리)
    ├── clipboard/
    │   ├── listener_windows.go         # winloop 창에 AddClipboardFormatListener
    │   ├── sensitive_windows.go        # 민감 포맷 검사
    │   ├── debouncer.go
    │   └── debouncer_test.go
    ├── tray/
    │   ├── tray_windows.go             # 트레이 아이콘·메뉴·풍선 표시 (winloop 창 사용)
    │   └── icon.go
    ├── notifier/
    │   ├── notifier.go                 # Notifier 인터페이스
    │   ├── balloon_windows.go          # MVP: tray 풍선 사용
    │   └── toast_windows.go            # 고도화: WinRT Toast
    ├── ipc/
    │   ├── pipe_windows.go             # SID 기반 파이프, 현재 사용자 DACL, 3초 재시도 연결
    │   ├── message.go                  # 한 줄 JSON 프로토콜 (v, op), 64KB 상한
    │   ├── protocol_windows.go         # HKCU clipwget:// 스킴 및 AppUserModelID 등록
    │   ├── uri.go                      # clipwget:// 액션 URI 파서
    │   └── *_test.go
    └── webui/
        ├── server.go                   # 127.0.0.1:0, CSRF + Host 검증, 하트비트 + 5분 유예 종료
        ├── server_test.go
        └── static/                     # go:embed 보관함 페이지
```

---

## 3. 패키지 간 핵심 인터페이스 계약 (Contracts)

### 3.1 도메인 타입 (`internal/media`)
```go
type LinkType string

const (
	LinkUnknown       LinkType = ""
	LinkSocialTwitter LinkType = "SOCIAL_TWITTER"
	LinkStreamHLS     LinkType = "STREAM_HLS"
	LinkDirectMedia   LinkType = "DIRECT"
)

// 중복 판정 단위 (설계 §5)
type Key struct {
	Dedupe string // x:<status_id> 또는 정규화 URL의 SHA-256
	Index  int    // 소셜 1-based (단일 영상도 1), HLS/직링크 0
}

// router 결과
type Source struct {
	Type       LinkType
	URL        string            // 사용자가 복사한 그대로 (실제 요청용)
	Headers    map[string]string // DevTools cURL 헤더 (메모리 전용)
	MediaIndex int               // 0 = 지정 안 함(게시물 전체), N ≥ 1 = 해당 순번만
	NeedsProbe bool              // #EXTM3U 프로빙 필요 여부
}

// extractor 결과 = 다운로드 후보 1건
type Item struct {
	Key            Key
	Type           LinkType
	SourceURL      string            // 원본 (소셜 재추출용)
	DisplayURL     string            // 사용자정보·쿼리·fragment 제거
	StreamURL      string            // 최종 MP4 또는 비디오 .m3u8
	AudioURL       string            // 분리 오디오 .m3u8 (없으면 "")
	Headers        map[string]string // 전달 정책(설계 §4.2-4) 적용 후 헤더, 영속 금지
	Author         string
	Title          string
	ThumbnailURL   string
	Resolution     string
	PostedAt       time.Time         // 없으면 감지 시각
	MediaCount     int               // 게시물의 미디어 수 (_N 접미사 판단용)
	SuggestedName  string            // naming 제안 (최종 이름은 Fetch에서 확정)
	RequiresFFmpeg bool
}

var (
	ErrLiveStream   = errors.New("live stream not supported")
	ErrDRM          = errors.New("drm protected stream not supported")
	ErrAuthRequired = errors.New("stream requires authentication")
	ErrExpired      = errors.New("item expired")
)
```

### 3.2 엔진 인터페이스
```go
// extractor
type Extractor interface {
	Match(s media.Source) bool
	Extract(ctx context.Context, s media.Source) ([]media.Item, error)
}

// history: 완료 이력만 영속 (헤더 저장 금지)
type Store interface {
	Find(k media.Key) (Record, bool, error)
	Upsert(r Record) error
	List(limit int) ([]Record, error)
	Delete(id int64) error
	Close() error
}
```

### 3.3 유스케이스 / 데몬 코어
- `pipeline.Pipeline` (`Resolve`, `Fetch`) — 설계 §2.3
- `app.Commands`, `ItemRef` — 설계 §2.5
- `inbox`는 `media.Item`만 알고 `history`·`router`·`extractor`를 import하지 않음

```go
type Inbox interface {
	SessionID() string
	Upsert(it media.Item, now time.Time) (Item, Change) // Change: Ignored / Added / Refreshed / InProgress
	Get(id uint64) (Item, bool)
	List() []Item
	MarkDownloading(id uint64) error // PENDING/FAILED에서만 성공
	MarkCompleted(id uint64, savedPath string) error
	MarkFailed(id uint64, reason string) error
	Remove(id uint64) error          // DOWNLOADING이면 거부
}

type Notifier interface {
	NotifyDetected(v app.ItemView) error
	NotifyCompleted(v app.ItemView) error
	NotifyFailed(v app.ItemView) error
	NotifyInfo(title, message string) error
	ClearAll() error
}
```

### 3.4 CLI 및 실행 인터페이스
| 명령어 | 바이너리 | 동작 설명 |
| :--- | :--- | :--- |
| `clipwgetw` | GUI | 트레이 데몬 실행 (이미 실행 중이면 `focus` 전달 후 종료) |
| `clipwgetw "clipwget://..."` | GUI | 알림 액션 처리 (파이프 전달, 데몬 없으면 데몬으로 승격 + 만료 안내) |
| `clipwget get <url \| curl 명령> [--force]` | 콘솔 | 데몬 없이 `pipeline`으로 즉시 다운로드 (수집함 미사용, 완료 이력 기록) |
| `clipwget watch` | 콘솔 | 클립보드 감지·`Resolve` 결과 실시간 출력 (민감 값 마스킹, 다운로드 안 함) |
| `clipwget list` | 콘솔 | 데몬 수집함(`list` op) + 최근 완료 이력 출력 |
| `clipwget history` | 콘솔 | 데몬에 `open-history` 요청 (데몬 없으면 안내) |
| `clipwget setup-ffmpeg` | 콘솔 | FFmpeg 진단 및 설치 |

---

## 4. 단계별 구현 로드맵 및 체크리스트

각 Step 완료 시 `gofmt`, `go vet ./...`, `go test ./...` 통과를 확인한 뒤 다음 단계로 넘어갑니다.

### Step 1: 모듈 초기화, 메모리 스파이크, 기반 유틸리티 (`media`, `config`, `naming`, `router`)
- [ ] `go mod init github.com/jinhoOps/clipwget` 및 패키지 골격 생성
- [ ] **메모리 스파이크** (`tools/memspike`): ① 메시지 루프 + 트레이 아이콘만, ② ①에 `modernc.org/sqlite` DB 연결 추가 — 두 상태의 작업 집합을 측정해 이 문서 §6에 기록. SQLite가 목표(10~15MB)를 크게 넘기면 `history`를 JSON Lines로 결정 (설계 §10)
- [ ] `internal/media`: 타입·오류 정의, `key.go` — 소셜 키, URL 정규화, 호스트별 인증 파라미터 목록, 표시용 URL (설계 §5)
- [ ] `internal/config`: 기본 다운로드 폴더(`%USERPROFILE%\Downloads\ClipWget`), 앱 데이터 폴더, 포터블 모드, JSON 로드/저장
- [ ] `internal/naming` (설계 §6):
  - [ ] 소셜 포맷 — URL·해시태그 제거, 개행 정제, 40룬 축약, 빈 본문 폴백, 로컬 시간, `MediaCount ≥ 2`일 때만 `_N`
  - [ ] 스트림 포맷 및 문맥 매칭 (3분 + 연관 근거 둘 다 필요)
  - [ ] 금지 문자, 제어 문자, 예약어, 후행 마침표/공백, 260자 단축
- [ ] `internal/router` (설계 §4):
  - [ ] x.com 계열 URL 및 `/video/N` 파싱 (일반 URL → `MediaIndex 0`, `/video/1` → `1`)
  - [ ] `.m3u8` 정규식 감지, 스트림 의심 URL 구분
  - [ ] `curl.go`: bash / cmd 토크나이저, `-H`/`--header`/`-b`/`--cookie`, 헤더 개행 입력 거부
  - [ ] 테이블 기반 단위 테스트 (일반 URL과 `/video/1`이 같은 `Key`로 귀결되는지 포함)
- **DoD**: `go test ./internal/media ./internal/config ./internal/naming ./internal/router` 통과, 메모리 측정 결과 기록

---

### Step 2: 추출 엔진 (`extractor`)
- [ ] `extractor.go`: `Extractor` 인터페이스, 안전 HTTP 클라이언트 (사설 IP 차단 다이얼러, `io.LimitReader` 크기 상한·타임아웃 — 설계 §7.4)
- [ ] `twitter.go`: fxtwitter 파서(최고 비트레이트, 다중 미디어 1-based 순번, `MediaCount`, webp 썸네일), vxtwitter 폴백, 방어적 파싱
- [ ] `hls.go` (설계 §4.2):
  - [ ] `#EXTM3U` 프로빙 (Range 무시 응답에도 256바이트 상한)
  - [ ] Master 파싱, 최고 화질 선택, `min(W,H)` 라벨, 분리 오디오 그룹 매칭(`DEFAULT=YES` 우선), 상대 URI 해석
  - [ ] Media Playlist 판정: `ENDLIST` 없음 → `ErrLiveStream` (종료된 `EVENT` 허용), `SAMPLE-AES`·DRM `KEYFORMAT` → `ErrDRM`
  - [ ] **헤더 전달 정책**: 모든 URI 호스트를 수집해 eTLD+1 비교 → 같은 사이트면 `Cookie`/`Authorization` 유지, 아니면 제거 (설계 §4.2-4)
- [ ] `direct.go`: `.mp4`, `.webm`, `.webp` 직링크 (순번 0)
- [ ] 테스트: Master/Media/분리오디오/상대경로/라이브/종료된 EVENT/DRM/AES-128/fMP4/교차 사이트 세그먼트/크기 초과 응답
- **DoD**: `go test ./internal/extractor/...` 통과 (외부 네트워크 없이 `testdata/` 픽스처만으로 결정론적)

---

### Step 3: 다운로드 엔진, 공용 파이프라인, `clipwget get` (`lock`, `history`, `ffmpeg`, `downloader`, `pipeline`)
- [ ] `internal/lock` (설계 §7.3):
  - [ ] 이름 있는 뮤텍스 래퍼 (`Local\ClipWget-job-<키 해시>`, `Local\ClipWget-daemon`)
  - [ ] 작업 폴더 생성 + `.owner` 표식 유지, 시작 시 안전 청소 (소유 프로세스가 끝난 폴더만)
  - [ ] `MoveFileEx` 덮어쓰기 금지 확정 + ` (2)`, ` (3)` 재시도 — **`os.Rename` 사용 금지**
  - [ ] 테스트: 대상 파일이 이미 있을 때 덮어쓰지 않음, 사용 중인 작업 폴더는 청소하지 않음
- [ ] `internal/history` (설계 §7.2): SQLite 구현 (WAL, `busy_timeout=5000`, `MaxOpenConns(1)`, upsert, `user_version`), 두 연결 동시 쓰기 테스트. Step 1 결과에 따라 JSON Lines 대체
- [ ] `internal/ffmpeg` (설계 §8.1): `Resolve()` 탐색, 고정 ZIP 다운로드 → SHA-256 검증 → `ffmpeg.exe` 추출, 가짜 ZIP으로 일치/불일치 테스트
- [ ] `internal/downloader`:
  - [ ] `http.go`: 헤더 지원 스트리밍 다운로드 (지정 경로에 기록), 썸네일 캐시 (인증 헤더 미전달)
  - [ ] `ffmpeg.go` (설계 §8.2): 입력별 `-headers`, 분리 오디오 `-map`, `-f mp4`, `-protocol_whitelist http,https,tcp,tls,crypto`, `aac_adtstoasc` 미지정, `HideWindow` + `CREATE_NO_WINDOW`, `exec.CommandContext` 취소
  - [ ] `ffmpeg_test.go`: `httptest.Server`로 미니 TS / fMP4 / AES-128 / 분리 오디오 HLS를 서빙해 결합 검증. FFmpeg 없으면 `t.Skip` (단, 릴리스 게이트는 §5.3)
- [ ] `internal/pipeline` (설계 §2.3, §7.3): `Resolve`, `Fetch` 8단계 구현
  - [ ] 테스트: 같은 키 `Fetch` 동시 2회 → 1회만 다운로드되고 다른 쪽은 `AlreadyCompleted`, 이력 쓰기 실패 시 파일 보존 + `HistoryErr`, 취소 시 작업 폴더 삭제
- [ ] `cmd/clipwget`: `get` (`pipeline` 사용, 진행률 출력, `--force`), `setup-ffmpeg`
- **DoD**: `go test ./internal/lock ./internal/history ./internal/ffmpeg ./internal/downloader ./internal/pipeline` 통과 + 실제 x.com 링크와 DevTools cURL로 `clipwget get` 성공 (수동, 날짜·결과 기록)

---

### Step 4: UI 스레드, 클립보드 감지, `clipwget watch` (`winloop`, `clipboard`)
- [ ] `internal/winloop` (설계 §2.4): `LockOSThread` UI 스레드, **숨겨진 최상위 창**(HWND_MESSAGE 아님), 메시지 핸들러 등록 API, `PostMessage` 기반 작업 전달
- [ ] `internal/clipboard`:
  - [ ] `debouncer.go`: 시퀀스 번호·최근 텍스트 기반 150ms 디바운스, 단위 테스트
  - [ ] `sensitive_windows.go`: 민감 포맷 검사 (설계 §11)
  - [ ] `listener_windows.go`: winloop 창에 `AddClipboardFormatListener`, `OpenClipboard` 재시도 백오프, 256KB 초과 무시
- [ ] `cmd/clipwget`: `watch` — 감지 → `pipeline.Resolve` 결과 출력, `Cookie`/`Authorization`/쿼리 마스킹
- **DoD**: `go test ./internal/clipboard` 통과 + `clipwget watch`에서 x.com·m3u8·cURL 감지, 비밀번호 관리자 복사는 무시 (수동)

---

### Step 5: 트레이 데몬 MVP (`inbox`, `app`, `tray`, `notifier/balloon`, `ipc` 파이프, `cmd/clipwgetw`)
- [ ] `internal/inbox` (설계 §7.1): 메모리 수집함, `SessionID`, `Upsert` 5초 쿨다운·갱신, `MarkDownloading` 조건부 전이, 200건 상한(`DOWNLOADING` 제외), fake clock 테스트
- [ ] `internal/app` (설계 §2.4, §2.5, §7.1):
  - [ ] `Commands` 구현, 워커 2개 + 대기 큐 200, `Resolve` 동시 4건 제한
  - [ ] 중복 정책 판정 (수집함 + 완료 이력 조회), 문맥 1건(3분) 보관
  - [ ] `ItemChanged` 이벤트 발행
  - [ ] 종료 순서: 신규 접수 중단 → HTTP·FFmpeg 취소 → 작업 폴더 정리 → `Notifier.ClearAll()`
  - [ ] 테스트: 같은 항목 다운로드 중복 호출 → 작업 1건, 다운로드 중 `Remove` 거부, 종료 중 신규 요청 거부
- [ ] `internal/tray`: 아이콘, 메뉴(`수집함 (N건)` 하위 항목별 다운로드, `다운로드 폴더 열기`, `감지 일시정지`, `자동 다운로드`, `FFmpeg 상태`, `종료`), 풍선 표시, `TaskbarCreated` 시 아이콘 재등록
- [ ] `internal/notifier`: 인터페이스 + `balloon_windows.go` (클릭 시 마지막 알린 항목 `Commands.Download`)
- [ ] `internal/ipc` 파이프 (설계 §9.3): SID 이름, 현재 사용자 DACL, 한 줄 JSON 프로토콜(`focus`, `list`), 64KB 상한, 3초 재시도 연결, 프로토콜 파서 테스트
- [ ] `cmd/clipwgetw`: `Local\ClipWget-daemon` 뮤텍스 획득 → 데몬 실행, 실패 시 `focus` 전달 후 종료
- [ ] `cmd/clipwget`: `list`
- **DoD**: 트레이 실행 → 복사 감지 → 수집함 → 풍선 클릭 또는 트레이 메뉴로 다운로드 완료. 앱 재실행 시 **수집함은 비어 있고 완료 이력은 유지**. 탐색기 재시작 후 트레이 아이콘 복구

---

### Step 6: 커스텀 프로토콜 & 인터랙티브 Toast (`ipc/uri`, `ipc/protocol`, `notifier/toast`)
- [ ] **WinRT 스파이크**: `go-ole` / `winrt-go`로 버튼 Toast 1건을 띄우는 최소 예제 → 라이브러리 선정 (`CGO_ENABLED=0` 확인). 실패 시 이 Step을 보류하고 Step 7로 진행
- [ ] `uri.go`: `download` / `open-folder` / `open-history` 파서, 알 수 없는 액션·잘못된 파라미터 거부 테스트
- [ ] `protocol_windows.go`: `HKCU\Software\Classes\clipwget` → `clipwgetw.exe "%1"`, `AppUserModelId\ClipWget.App` 등록/갱신
- [ ] 파이프 프로토콜에 `download`, `open-folder`, `open-history` op 추가
- [ ] `toast_windows.go`: 버튼 Toast, `ToastNotificationHistory.Clear`, 초기화 실패 시 풍선 폴백
- [ ] `cmd/clipwgetw`: 프로토콜 인자 처리 — 데몬 있으면 전달, 없으면 승격 + "만료된 항목", `SessionID` 불일치 시 `ErrExpired` 안내
- **DoD**: URI·IPC 왕복 테스트 통과, Toast 버튼으로 다운로드, 재시작 후 이전 Toast 제거 및 남은 알림 클릭 시 "만료된 항목"

---

### Step 7: 웹 보관함(`webui`), 마감 및 문서
- [ ] `server.go` (설계 §9.4): `127.0.0.1:0`, CSRF + `Host` 검증, API(`/api/inbox`, `/api/history`, `/api/download`, `/api/open-folder`, `/api/remove`, `/api/history/delete`, `/api/heartbeat`)는 모두 `app.Commands` 호출, 페이지별 하트비트 + 5분 유예 종료, 기존 서버 재사용
- [ ] 기본 브라우저 실행, 실패 시 주소 알림 + 클립보드 복사
- [ ] `static/`: 다크모드 보관함 (수집함 탭 + 휘발 안내, 완료 이력 탭, 썸네일 카드, 2초 폴링)
- [ ] `server_test.go`: 잘못된 Host·토큰 거부, 하트비트 만료 후 종료, 재연결
- [ ] 트레이 `보관함 열기`, `clipwget history` 연결
- [ ] 문서: `README.md`(설치·빌드·사용법·휘발성 수집함·지원 범위 밖), `CLAUDE.md`·`AGENTS.md`(패키지 구조·명령어)
- **DoD**: `go build ./...`, `go vet ./...`, `go test ./...` 통과 및 §5 점검 완료

---

## 5. 검증 매트릭스 (Verification Plan)

### 5.1 자동화 검증 명령어 (PowerShell)
```powershell
# 코드 포맷 및 정적 검사
gofmt -s -l .        # 출력이 비어 있어야 함 (수정은 gofmt -s -w .)
go vet ./...

# 전체 테스트 (FFmpeg 미설치 시 결합 테스트는 Skip — 릴리스 판정에는 불충분)
$env:CGO_ENABLED = "0"; go test -v ./...

# 콘솔 CLI 빌드
$env:CGO_ENABLED = "0"; go build -trimpath -ldflags="-s -w" -o clipwget.exe ./cmd/clipwget

# GUI 트레이 데몬 빌드
$env:CGO_ENABLED = "0"; go build -trimpath -ldflags="-s -w -H=windowsgui" -o clipwgetw.exe ./cmd/clipwgetw
```

### 5.2 Windows 수동 점검 체크리스트
1. **소셜 감지**: `https://x.com/<user>/status/<id>` 복사 시 1초 이내 알림 및 `@작성자 본문_날짜.mp4` 제안
2. **m3u8 & cURL**: 쿼리 토큰 `.m3u8`, `Copy as cURL (bash)`/`(cmd)` 복사 시 최고 해상도 판별 및 오디오 포함 `.mp4`
3. **라이브/DRM 거부**: 라이브 방송 m3u8 복사 시 거부 메시지
4. **세션 내 유지**: 알림을 무시한 뒤 트레이/보관함에서 `PENDING` 항목 다운로드
5. **휘발**: 앱 종료(또는 재부팅) 후 수집함 비어 있음, 완료 이력 유지, 이전 알림 클릭 시 "만료된 항목"
6. **중복 방지**: 받은 링크 재복사 시 "이미 다운로드된 미디어입니다" + `[폴더에서 보기]`
7. **민감 콘텐츠**: 비밀번호 관리자 복사값이 `clipwget watch`에 나타나지 않음
8. **동시성**: 같은 링크를 트레이와 `clipwget get`으로 동시에 받으면 한쪽만 다운로드, 파일 덮어쓰기 없음
9. **실패 상황**: 네트워크 단절, 디스크 부족, 만료 URL, 다운로드 중 작업 관리자로 강제 종료 후 재시작 시 작업 폴더 청소
10. **탐색기 재시작**: `explorer.exe` 재시작 후 트레이 아이콘 복구
11. **브라우저**: 기본 브라우저 변경, 실행 실패, 다중 탭, 최소화, 절전 복귀 후 보관함 동작
12. **메모리**: 빈 수집함 / 200건 / 다운로드 직후 작업 집합 측정 (브라우저·FFmpeg 분리), 보관함 닫은 뒤 서버 종료 확인

### 5.3 릴리스 게이트
- FFmpeg Skip만으로 완료 처리하지 않습니다. 고정 버전 FFmpeg가 설치된 Windows에서 **TS / fMP4 / AES-128 / 분리 오디오** 결합 테스트를 반드시 통과합니다 (`aac_adtstoasc` 자동 삽입 확인 포함).
- 외부 API(fxtwitter/vxtwitter) 수동 점검 날짜·결과와 메모리 측정 환경(OS 빌드, Go 버전)을 §6에 기록합니다.

---

## 6. 측정 및 점검 기록

| 날짜 | 항목 | 환경 | 결과 |
| :--- | :--- | :--- | :--- |
| | Step 1 메모리 스파이크 ① 트레이만 | | |
| | Step 1 메모리 스파이크 ② + SQLite | | |
