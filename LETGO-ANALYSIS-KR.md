# let-go 전수조사 & 활용 분석 (한국어)

> 이 문서는 `let-go` 저장소를 전수조사하고, 설치/사용법·정체성·활용처·수익화
> 아이디어까지 정리한 분석 노트입니다. 대화 내용을 문서로 옮긴 것이라 저장소의
> 코드나 빌드에는 영향을 주지 않습니다.

## 관련 링크

| 대상 | 주소 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/let-go |
| 원본 저장소 (upstream) | https://github.com/nooga/let-go |
| 릴리스 (미리 빌드된 바이너리) | https://github.com/nooga/let-go/releases |
| 브라우저 REPL (WASM 데모) | https://nooga.github.io/let-go/ |
| Discord `#let-go` | https://discord.gg/Ky535CQ9pj |
| jank Clojure 테스트 스위트 | https://github.com/jank-lang/clojure-test-suite |
| lgcr (let-go로 만든 컨테이너 런타임) | https://github.com/nooga/lgcr |
| xsofy (let-go로 만든 로그라이크) | https://github.com/nooga/xsofy |
| lgx (프로젝트/의존성 관리) | https://github.com/abogoyavlensky/lgx |
| Babashka pods | https://github.com/babashka/pods |
| paserati (같은 저자의 Go제 TS 런타임) | https://github.com/nooga/paserati |

---

## 1. 한 줄 정의

**`let-go`는 Go로 작성된 Clojure 방언 구현체다.** 리더 → 바이트코드 컴파일러 →
스택 VM 구조이며, JVM 없이 단일 바이너리로 동작한다. 선택적 IR 경로와 Go AOT
백엔드도 함께 제공한다. 라이선스는 MIT.

## 2. 저장소 규모 (직접 측정)

| 항목 | 수치 |
|---|---|
| Go 소스 | 568개 파일 / 136,640줄 |
| `.lg` (Clojure) 소스 | 388개 파일 / 46,463줄 |
| `docs/` 문서 | 52개 |
| 테스트 | `test/` 아래 300개 이상 |
| 합계 | 약 18만 줄 |

## 3. 디렉터리 구조

### `pkg/` — 엔진 본체

| 경로 | 크기 | 역할 |
|---|---|---|
| `pkg/rt` | 3.3M | 런타임/표준 라이브러리 (http, json, os, syscall, term, pods, async) |
| `pkg/vm` | 1.2M | 스택 기반 가상머신, 값 표현, 영속 자료구조 |
| `pkg/ir` | 704K | 중간표현(IR), 최적화 패스, Go 코드 생성(gogen) |
| `pkg/compiler` | 288K | Clojure 소스 → 바이트코드 컴파일러 (기본 경로) |
| `pkg/bytecode` | 228K | `.lgb` 포맷: 인코더/디코더/압축/스트립 |
| `pkg/cli` | 84K | `lg` 커맨드라인 구현 전체 |
| `pkg/api` | 68K | Go 프로그램 임베딩 API (`NewLetGo`, `Def`, `Run`) |
| `pkg/genmanifest` | 92K | 생성물 매니페스트 |
| `pkg/wasmhost` | 32K | 브라우저/WASM 호스트 |
| `pkg/resolver` | 32K | 네임스페이스 해석 |
| `pkg/nrepl` | 20K | nREPL 서버 (CIDER/Calva/Conjure) |
| `pkg/bundle` | 40K | 단일 실행파일 번들링 |

### 실행 파이프라인

```
.lg 소스 → 리더 → 컴파일러 → 바이트코드(.lgb) → 스택 VM → 실행
                      └→ IR → Go 소스 생성 → 네이티브 컴파일 (AOT)
```

컴파일 경로는 세 가지다.

1. 기본 바이트코드 VM (`pkg/compiler`)
2. IR 경로 (`*ir-compile*`)
3. Go AOT 백엔드 (`-tags gogen_ir`)

`AGENTS.md`에 따르면 셋 중 하나를 바꾸는 변경은 나머지 둘에 대한 입장을 함께
밝혀야 한다.

### `pkg/rt/core/` — 자기호스팅 표준 라이브러리

`core.lg`(2,969줄)를 비롯해 `string.lg`, `set.lg`, `walk.lg`, `edn.lg`,
`pprint.lg`, `test.lg`, `async.lg` 등 표준 라이브러리를 let-go 자신으로 작성했다.

> 주의: `pkg/rt/core/**/*.lg` 또는 `pkg/ir/ir_*.lg`를 수정하면 `make generate`를
> 돌려야 반영된다. 재생성 결과물은 원본 수정과 같은 커밋에 포함해야 하며,
> `make check-generated`가 이를 검증한다. `make build`는 검증하지 않는다.

### `cmd/` — 보조 도구 11종

`lgbgen`(바이트코드 생성), `lgbstat`(분석), `lginterop`(Go 패키지 래핑 생성),
`perf-page`(성능 페이지), `bench-ratchet`(성능 회귀 방지), `check-generated`,
`hoist-natives`, `lgprimgen`, `bootprobe`, `lg-runtime`.

### `test/` — 300개 이상

`.lg` 언어 테스트, `clojure-test-suite/`(jank 공식 스위트), `gold`/`gold-aot`
골든 테스트, `benches/` 벤치마크, Go 테스트가 패키지 옆에 위치.

### `docs/` — 52개 문서

- `guide/` — 사용자용: 설치, 사용법, 임베딩, Go interop, nREPL, WASM, pods, 포터빌리티
- `design/` — 설계: VM 최적화, 값 표현, IR 로워링, 런타임 이미지, Go AOT, I/O 호스트 분리
- `perf/` — 성능 baseline과 회귀 방지 래칫
- 루트 — `master-plan.md`, `contribution-policy.md`, `contributor-workflow.md` 등

모든 문서가 `status` / `last-verified` frontmatter를 가지며
`scripts/docs_frontmatter_hook.py`가 검사한다.

### `examples/` — 실전 예제

`server.lg`(Ring 스타일 HTTP 서버), `goroutines.lg`, `concurrent-primes.lg`,
`mandelbrot.lg`, `gol.lg`, `browser-inspector/`, `host-eval/`,
`malli-on-let-go/`, `ys-on-let-go/`, `aot/`.

### `.github/workflows/` — CI 10종

빌드/테스트/린트/CodeQL/릴리스 + 성능 전용 워크플로우 6종
(`perf-pr`, `perf-pr-repeat`, `perf-timeline`, `perf-wasm`, `perf-backfill`,
`perf-release-baseline`).

### 이 포크에 추가된 것

- `CLAUDE.md` — 페르소나 가이드 (커밋 `261a1f5`)
- `AGENTS.md` — AI 에이전트용 기여 가이드
- `.claude/skills/comment-lint/`, `.claude/skills/docs-status/`
- `.agents/skills/` — 동일 스킬 미러

## 4. 직접 빌드/실행 검증 결과

```bash
$ go build -o lg .
$ ls -lh lg
21M

$ ./lg -version
lg 0.0.0-20260918032151-d51816bc084a (d51816b)
lgb: format 2 (default write), 3 (max)
capabilities: CapOpcodeSet (0x1, since v1.12.0)

$ ./lg -e '(+ 1 2)'
3
$ ./lg -e '(map inc [1 2 3])'
(2 3 4)
$ ./lg -e '(take 10 (map #(* % %) (range)))'
(0 1 4 9 16 25 36 49 64 81)

$ time ./lg -e '(println "hi")'
real  0m0.009s
```

콜드 스타트 9ms를 실제로 확인했다. README의 벤치마크 표(Apple M1 Pro 기준)는
다음과 같다.

| | let-go | let-go AOT | babashka | Clojure JVM |
|---|---|---|---|---|
| 바이너리 크기 | 13MB | 18MB | 68MB | 304MB (JDK) |
| 시작 시간 | 11.1ms | 10.7ms | 20.4ms | 364ms |
| 유휴 메모리 | 15.2MB | 15.2MB | 27.0MB | 97.7MB |

AOT 효과: `fib(35)` 2.42초 → 0.11초, `tak` 2.40초 → 94ms.

## 5. 설치 및 사용법

### 설치

```bash
brew install nooga/tap/let-go          # Homebrew (macOS/Linux)
go install github.com/nooga/let-go@latest   # Go 1.26+
go build -o lg .                        # 이 저장소에서 직접 빌드 (검증됨)
```

미리 빌드된 바이너리는 Releases에서 Linux / macOS / Plan 9용을 받을 수 있다.
Windows 공식 빌드는 없다(WSL 사용).

### 실행

```bash
lg                          # REPL
lg -e '(+ 1 1)'             # 한 줄 평가
lg myfile.lg                # 파일 실행
lg myfile.lg Alice Bob      # 인자 전달 (*command-line-args*)
lg -r myfile.lg             # 실행 후 REPL 진입
```

### 배포

```bash
lg -c app.lgb app.lg        # 바이트코드 컴파일
lg -b myapp app.lg          # 단일 실행파일 번들
lg -b myapp -z app.lg       # DEFLATE 압축
lg -strip -b myapp app.lg   # 소스맵을 myapp.debug로 분리
lg -w site app.lg           # WASM 웹앱 (~6MB 단일 index.html)
```

### 에디터 연동

```bash
lg -n                       # nREPL, 기본 포트 2137
lg -n -p 7888
```

`.nrepl-port` 파일을 작업 디렉터리에 써서 에디터가 자동 탐색한다.
CIDER(Emacs), Calva(VS Code), Conjure(Neovim) 지원.

### 주요 플래그

| 플래그 | 설명 |
|---|---|
| `-e` | 표현식 평가 |
| `-c <out>` / `-b <out>` / `-w <dir>` | 바이트코드 / 단일 바이너리 / WASM |
| `-w-shell` | `xterm`(기본) / `none` / 커스텀 HTML 템플릿 |
| `-w-host-eval` | JS에서 `LetGoHost.eval(code)` 호출 허용 |
| `-w-wasm` | `inline`(기본) / `external` |
| `-n`, `-p` | nREPL 서버 / 포트 |
| `-z` | 번들 압축 |
| `-strip`, `-debug-output` | 디버그 컴패니언 분리 |
| `-source-paths`, `-resource-paths` | 네임스페이스/리소스 검색 경로 |
| `-d` | VM 디버그 모드 |

### Go 임베딩

```go
c, _ := api.NewLetGo("myapp")
c.Def("greet", func(name string) string { return "Hello, " + name })
v, _ := c.Run(`(greet "world")`)   // "Hello, world"
```

Go 구조체는 레코드로 왕복하고(`vm.RegisterStruct[T]`), Go 채널은 let-go 채널로
그대로 쓰이며, Go 함수는 let-go에서 호출 가능하다. `WithStdout` / `WithStderr` /
`WithEmit` / `WithKeySource` 옵션으로 I/O를 호스트가 가로챌 수 있다.
`lg_no_http` 빌드 태그로 `net/http`(및 `crypto/tls`, `crypto/x509`)를 바이너리에서
아예 제거할 수 있다.

## 6. 자주 나오는 질문 정리

### 플러그인인가, 스킬인가, MCP인가?

셋 다 아니다. **프로그래밍 언어 구현체(런타임 + 컴파일러 + VM)** 이다.
다만 이 저장소에는 곁다리로 Claude Code용 스킬 두 개
(`.claude/skills/comment-lint`, `.claude/skills/docs-status`)와 `CLAUDE.md`,
`AGENTS.md`가 들어 있어서 혼동하기 쉽다. 본체와는 별개다.

### API 토큰이 필요한가?

**필요 없다.** `pkg/`와 `cmd/` 전체를 `api[_-]?key|api[_-]?token|openai|anthropic|bearer`
패턴으로 검색했을 때 매칭이 0건이었다. 로그인·계정·API 키·텔레메트리 모두 없고,
빌드 이후에는 완전 오프라인으로 동작한다. 라이선스는 MIT라 상업적 이용·수정·
재배포가 자유롭다.

예외는 (1) 설치 시 다운로드, (2) 사용자가 작성한 프로그램이 외부 API를 부를 때,
(3) Babashka pod을 처음 내려받을 때뿐이다.

### 왜 GitHub에서 주목받는가

> 참고: 이 분석 세션은 저장소 접근이 `bmshin94/let-go`로 제한되어 있어 원본
> 저장소의 실시간 스타 수는 확인하지 않았다. 아래는 저장소 내용 기반 분석이다.

1. **"JVM 없는 Clojure"** — Clojure의 오랜 약점인 시작 시간을 364ms → 9ms로 해결
2. **Go + Lisp 조합** — README에 "Make it legal to write Clojure at your Go dayjob"
3. **유머와 밈** — `let-go` = λ-gopher 말장난, "2021년에 농담으로 시작했는데 쓸모있어짐"
4. **검증 가능한 성취** — jank 테스트 5,621/5,621 통과(실패 0), 벤치마크 정직 공개
5. **이식성 스토리** — 브라우저(WASM), Plan 9, reMarkable 2에서 동작
6. **실제 제품** — `lgcr`(컨테이너 런타임), `xsofy`(게임)를 이걸로 만듦
7. **AOT 성능** — Clojure를 Go 소스로 낮춰 20배 이상 가속

### 로컬 에이전트 구축에 도움이 되는가

**"두뇌"로는 부적합, "손발/샌드박스"로는 적합하다.**

도움이 되는 부분:

- LLM 생성 코드의 **화이트리스트 샌드박스**. `Def`한 함수만 접근 가능하고,
  `lg_no_http`로 네트워크 스택 자체를 제거할 수 있다.
- **코드가 곧 데이터** — LLM이 EDN/S-expression으로 툴 체인을 생성하면 파싱 없이 실행
- **초경량 워커** — 9ms 시작 + 15MB로 수백 개 동시 실행 가능
- **단일 바이너리 배포** — `requirements.txt` / `node_modules` 없음
- **동시성 내장** — `core.async` + 실제 고루틴

부족한 부분: LLM SDK 없음, MCP 구현체 없음, 벡터DB 클라이언트 없음,
LangChain류 프레임워크 없음, Clojure 러닝커브.

권장 아키텍처:

```
[Python/TS 에이전트 본체]  ← LLM 호출, 프롬프트 관리
          ↓ 생성 코드 전달
[let-go 샌드박스]          ← 안전한 실행, 화이트리스트 툴
          ↓
[Go 호스트 앱]             ← 실제 DB/API 접근
```

### React나 PHP로 만들 수 있는가

두 가지로 나눠서 봐야 한다.

**(1) React/PHP로 let-go를 재구현?** 기술적으로는 가능하지만(JS에는 ClojureScript,
Squint, Cherry가, PHP에는 Phel이 이미 있다) 의미가 없다. let-go의 가치는
9ms 시작 + 13MB 단일 바이너리인데, Node는 시작만 30~100ms에 `node_modules`가
필요하고 PHP는 단일 바이너리가 불가능하다.

**(2) React/PHP 프로젝트에 let-go를 붙이기?** 이게 실질적인 답이고, 잘 맞는다.

React — WASM 조합:

```bash
lg -w site app.lg --w-shell none --w-host-eval
```

```jsx
const result = window.LetGoHost.eval(userRule);
```

브라우저 내 룰 에디터, 인터랙티브 튜토리얼, 프라이버시 보존 데이터 변환기
(데이터가 서버로 나가지 않음), 노코드 수식 엔진, 샌드박스 플러그인 시스템에
적합하다. 번들이 약 6MB이므로 lazy-load나 `-w-wasm external`을 고려한다.

PHP — 사이드카 패턴 (직접 임베드는 불가, PHP 확장 모듈 없음):

1. `shell_exec()` 호출 — 9ms 시작이라 요청당 실행해도 체감 지연이 거의 없다
2. HTTP 마이크로서비스 — let-go가 `:9000`에서 서비스, PHP가 HTTP로 호출 (권장)
3. 프론트엔드 WASM — PHP는 페이지만 서빙, 계산은 브라우저에서

| 조합 | 방식 | 적합도 |
|---|---|---|
| Go + let-go | 네이티브 임베딩 (`pkg/api`) | 최상 |
| React / Next.js + let-go | WASM (`-w`) | 최상 |
| PHP / Laravel + let-go | `exec()` 또는 HTTP 사이드카 | 좋음 |
| JS / PHP로 let-go 재구현 | — | 비권장 |

## 7. 수익화 아이디어

대전제: **let-go 자체는 MIT라서 팔 수 없다. let-go를 재료로 그 위에 가치를 얹어
파는 구조여야 한다.**

### S급

**① 라이브 룰 엔진 SaaS** — 비즈니스 규칙을 코드 문자열로 DB에 저장하고
`pkg/api` 임베딩으로 평가. 기획자가 관리자 페이지에서 고치면 재배포 없이 즉시
반영된다.

```go
c.Def("order-total", order.Total)
c.Def("user-tier",   user.Tier)
discount, _ := c.Run(db.GetRule("discount_policy_v3"))
```

타겟: 이커머스(프로모션), 핀테크(신용평가/사기탐지), 물류(배송비), 게임(밸런싱),
SaaS(요금제/쿼터). 경쟁: Drools(JVM 필요), AWS EventBridge(락인),
LaunchDarkly(단순 플래그). 난이도 중, MVP 3~4개월, B2B 고단가.

**② AI 코드 실행 샌드박스** — LLM 생성 코드를 9ms에 격리 실행. 화이트리스트
`Def` + `lg_no_http` + 타임아웃/메모리 제한. 경쟁: E2B(Firecracker ~500ms),
Modal(컨테이너 ~1s), Deno Deploy(V8 ~50ms). 난이도 상, 4~6개월,
**보안 설계가 사업의 전부**다.

**③ React + WASM 인터랙티브 콘텐츠** — 백엔드 없이 브라우저에서 코드가 도는
사이트. 학습 플랫폼, 프라이버시 보존 데이터 도구(GDPR 유리), 문서 사이트용
플레이그라운드 위젯 B2B 납품. 난이도 하, 1~2개월, 초기 비용 거의 0.

### A급

**④ 교육 콘텐츠** — 리더→컴파일러→VM→IR→AOT 전 과정이 동작하는 18만 줄 교보재.
"Crafting Interpreters"의 Go 버전 시장이 비어 있다. 전자책/비디오/부트캠프/
기업 워크샵.

**⑤ CLI 도구 부티크** — 9ms 시작이 무기가 되는 영역(git hook 러너, 로그 분석,
데이터 변환, 빌드 오케스트레이터, 터미널 대시보드). 도구당 2~4주, 일회성 구매
또는 오픈코어.

**⑥ 컨설팅/개발 대행** — 임베딩 통합, JVM Clojure 마이그레이션, 성능 최적화,
커스텀 `lg` 빌드. 희소성 프리미엄이 있지만 전문성 구축에 6~12개월.

### B급

**⑦ MCP 서버 마켓플레이스** — 9ms/15MB/단일 바이너리는 MCP 서버 스펙으로 이상적.
단, MCP 구현체를 직접 만들어야 한다(2~4주).

**⑧ 엣지/IoT 런타임** — Plan 9와 reMarkable에서 도는 이식성이 곧 엣지 적합성.
하드웨어 파트너십과 긴 세일즈 사이클이 필요.

**⑨ 게임 모딩 런타임** — Lua보다 강력하고 샌드박싱이 쉽지만 업계의 Lua 관성이 크다.

### 추천 로드맵

| 단계 | 기간 | 내용 |
|---|---|---|
| Phase 0 | 1~2주 | CLI/HTTP 서버/임베딩 각 1개씩 만들어 감 잡기 |
| Phase 1 | 1~2개월 | ③ React+WASM 데모 공개 → 트래픽·신뢰 확보 (비용 ~0) |
| Phase 2 | 3~4개월 | ① 룰 엔진 SaaS MVP → Phase 1 유입자에게 베타 |
| Phase 3 | 6개월~ | ② AI 샌드박스로 확장(스택 재활용), ④ 교육으로 브랜딩, ⑥ 인바운드 컨설팅 |

### 리스크

| 리스크 | 내용 | 대응 |
|---|---|---|
| 버스 팩터 1 | 메인테이너가 사실상 1인 | 포크 후 직접 유지보수 각오 |
| 작은 생태계 | 라이브러리 거의 없음 | 필요한 건 직접 구현 |
| 고용 시장 없음 | 채용 공고가 없다 | 부업/자기사업 전제 |
| Clojure 인구 | 잠재 사용자 자체가 적음 | 고객이 Clojure를 몰라도 되게 설계 |
| 기술 리스크 | 버그를 직접 고쳐야 함 | Go 실력 필수 |

핵심 전략: **let-go를 파는 게 아니라, let-go로 만든 해결책을 판다.** 고객은
let-go가 무엇인지 몰라도 된다. "재배포 없이 규칙 바꾸기", "안전한 AI 코드 실행"
같은 고객의 문제를 파는 것이다.

## 8. 알려진 한계 (README 요약)

JVM Clojure의 드롭인 대체가 아니다. JAR을 로드하지 않으며 그럴 계획도 없다.

- 코디네이트된 STM / 비동기 agent 없음 (`ref`/`agent`는 atom 기반 별칭)
- `clojure.spec` 없음
- 청크되지 않은 lazy seq
- 커스텀 `*data-readers*` 없음
- `deftype`/`reify`의 JVM 호스트 interop 제한
- `subseq`/`rsubseq` 범위 질의 없음
- 실용주의적 수치 타워, 항상 블로킹하는 채널, 실제 고루틴 기반 `go` 블록,
  Java가 아닌 `re2` 정규식

전체 목록과 근거는 `docs/guide/clojure-compatibility.md`와
`docs/known-divergences.md`에 있다.

## 9. 기여 시 주의사항 (AGENTS.md 요약)

- `pkg/rt/core/**/*.lg` 또는 `pkg/ir/ir_*.lg` 수정 시 `make generate` 필수,
  재생성 결과물은 같은 커밋에 포함. `make check-generated`가 검증한다.
- 저장소 루트에 테스트 파일을 두지 않는다. `.lg` 테스트는 `test/`,
  Go 테스트는 해당 패키지 옆에.
- `docs/` 문서는 frontmatter 필수. `python3 scripts/docs_frontmatter_hook.py --check`
- 네이티브 등록은 `ns.Def`(`pkg/rt/lang.go`) 또는 `//lg:native` 마커 중 하나만.
- 세 컴파일 경로(기본 / `*ir-compile*` / `-tags gogen_ir`) 중 하나를 바꾸면
  나머지 둘에 대한 입장을 밝혀야 한다.
- 성능 주장은 `docs/perf/ratchet.md`를 거친다. 같은 머신 단일 실행은 근거가 아니다.
- 훅 설치: `prek install --install-hooks --hook-type pre-commit --hook-type pre-push`
- 머지 드라이버 등록: `make install-hooks` (클론마다 1회)
