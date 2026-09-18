# ZedBoard PQC 보안채널 가속기 — PR 연계 검토 (후속)

- **리뷰 일자:** 2026-09-17
- **리뷰어:** `fpga-code-reviewer` (역할 수행)
- **선행 문서:** [`2026-09-16-zedboard-pqc-secure-review.md`](2026-09-16-zedboard-pqc-secure-review.md) — 이 문서는 그 내용을 대체하지 않고, 대상 저장소의 **열린 PR 2건**을 교차 검토한 결과만 추가한다.
- **방법:** `external/zedboard-pqc-secure`를 재-clone(YAGNI 원칙, 캐시 미유지)한 뒤 `main` 기준 소스를 라인 단위로 재확인하고, GitHub REST API(`/pulls`, `/pulls/{n}`)로 PR 메타데이터·본문·머지 가능성을 조회했다. PR diff 자체(`*.diff`)는 이 세션에서 받지 않았으므로, 아래 판단은 **PR 본문의 서술과 main의 실제 코드를 대조한 결과**이며 PR 브랜치의 실제 코드는 별도 확인이 필요하다.

---

## 0. PR 현황

| # | 브랜치 | 상태 | Mergeable | 요지 |
|---|---|---|---|---|
| [#1](https://github.com/bobobomin/zedboard-pqc-secure/pull/1) | — | closed, **merged** (2026-08-21) | — | AXI4-Lite frontend 결함 4건 수정 |
| [#2](https://github.com/bobobomin/zedboard-pqc-secure/pull/2) | — | closed, **merged** (2026-08-30) | — | 100MHz rtl fix: BaseMul pipeline & replace modulo |
| [#3](https://github.com/bobobomin/zedboard-pqc-secure/pull/3) | — | closed, **merged** (2026-08-30) | — | Integrate/64session pr2 |
| [#4](https://github.com/bobobomin/zedboard-pqc-secure/pull/4) | `100MHz_rtl_fix` | **open** | **false (충돌)** | 64세션 설계 100MHz 타이밍 실패 수정 (Poly1305 r×5, carry chain, ChaCha20 반라운드, SHA3 팬아웃, AEAD 버스) |
| [#5](https://github.com/bobobomin/zedboard-pqc-secure/pull/5) | `2x_improvement` | **open** | **true** | BaseMul/NTT/INTT 파이프라인화 + Keccak 선행인출 + 핑퐁뱅크. 105,286 → 49,239 사이클 (2.27x), 100MHz 전체 블록디자인 타이밍 클로저 |

현재 README/`docs/OPTIMIZATION.md`가 보고하는 105,286사이클·WNS +0.105ns 기준선은 **PR #1~#3이 이미 반영된 `main`의 상태**다. #4, #5는 아직 `main`에 없다.

---

## A. PR #4 — 재검토 결과: 대부분 이미 `main`에 반영되어 있어 **stale/중복**으로 판단됨

PR #4 본문이 "기존 문제"로 지적한 4가지를 현재 `main`과 대조했다.

| PR #4가 주장하는 기존 문제 | `main`에서 실제 확인한 상태 | 판정 |
|---|---|---|
| Poly1305 `r×5`를 매 클럭 재계산 | `poly1305_fixed96.sv:147` — `S_IDLE`에서 `r5_limb[i] <= key_limb[i]*5`로 **1회만** 계산. 이미 수정됨 | **이미 반영** |
| Poly1305 자리올림 6개 64비트 덧셈 직렬(CARRY4 44단) | `poly1305_fixed96.sv:203-245` — `S_NORM0`~`S_NORM5` 6단계로 캐리 1스텝/클럭 분할. 이미 수정됨 | **이미 반영** |
| ChaCha20 라운드 1개에 32비트 덧셈 39단 | `chacha20_block.sv:35-51` — `quarter_round_half`로 반라운드 분할(`second_i`로 절반 선택), 40 스텝 FSM. 이미 수정됨 | **이미 반영** |
| SHA3 제어신호 1개가 1,600비트 스펀지 전체를 팬아웃 | `sha3_shake_stream.sv:32,70` — `control_state`/`absorb_active`에 `(* max_fanout = 64 *)` 지정, `sponge_state` 레지스터는 1벌만 존재(`keccak_f1600.sv`의 `state_reg`와 별개). 이미 수정됨 | **이미 반영** |
| AEAD 요청 버스 2,048비트가 512비트 payload를 감싸 계층 넘어 디먹스/먹스 | `aead_arbiter_4session.sv:29-38` — `req_data_i [2047:0]` vs 엔진 `engine_data_in [511:0]`. **여전히 존재** | **미반영** |

즉 PR #4가 다루는 5개 항목 중 4개는 **PR #2("100MHz rtl fix: BaseMul pipeline & replace modulo")가 이미 병합하면서 해결**한 것으로 보이고, PR #4는 그보다 앞선 커밋(`created_at 2026-08-31`, PR #2/#3 병합 하루 뒤)을 기준으로 만들어진 브랜치라 `main`과 diverge된 상태에서 conflict가 난 것으로 판단된다(`mergeable: false`가 이를 뒷받침).

**남은 유효 항목은 AEAD 2,048비트 버스 1건뿐**이며, 이는 이 저장소의 `aead_arbiter_4session.sv`가 **합성 closure 밖 파일**이라는 점(선행 리뷰 §F-2)까지 고려하면 우선순위가 낮다. `mlkem_secure_channel_complete_axi_top` 계층에서 실제로 쓰이는 세션 매니저(`aead_session_manager_bram.sv`) 경로의 버스 폭은 별도 확인이 필요하다.

**권장.** PR #4는 **닫거나 `main`에 rebase 후 diff를 재산출**해서 실제로 남는 변경이 있는지 다시 확인해야 한다. 현재 상태로 리뷰/머지 시간을 쓰는 것은 낭비다. `aead_arbiter_4session.sv`의 버스 폭 문제만 별도 이슈로 분리해 추적할 것을 권한다.

---

## B. PR #5 — 선행 리뷰 §C-4/G-13과 독립적으로 같은 결론에 도달했고, 더 나은 해법을 제시함

2026-09-16 리뷰의 **C-4 / G-13**은 정확히 이 문제를 지적했다: `mlkem_poly_accelerator.sv`의 BaseMul(9상태)/NTT(5상태)/INTT(7상태)가 계수쌍 1개당 파이프라인을 독점해 곱셈기가 대부분 유휴 상태라는 것. 그 리뷰는 BRAM 2포트 제약 때문에 **2사이클/연산**을 상한으로 보고 약 35% 감소(105,286 → 약 68,600)를 추정했다.

PR #5는 같은 문제를 **BRAM 뱅크를 주소 패리티로 두 개(`bank_a` 짝/홀)로 나누는 방법**으로 더 밀어붙여 사실상 **1사이클/버터플라이**에 근접시켰다고 주장한다(`NTT 17,928→3,704`, `INTT 26,632→4,772`, `BaseMul 9,232→1,104`). 이는 리뷰가 검토하지 않은 접근이며, 리뷰의 전제("완전 파이프라인은 4포트가 필요해 불가능")를 **뱅크 분할로 우회**한 것이므로 타당성이 높다. 같은 레이어 안에서 짝을 이루는 두 주소(`i`, `i+span`)가 항상 패리티가 반대라는 성질(선행 리뷰 §E-8이 이미 "NTT/INTT의 dual-port 동시 쓰기가 안전"함을 검증하며 확인한 것과 동일한 성질)을 이용한 것으로, 근거가 자연스럽다.

**교차 검증 포인트 (PR 브랜치 diff를 직접 받으면 반드시 확인할 것):**
1. 리뷰 §C-1/G-9가 지적한 `mlkem_poly_accelerator.sv:44`의 32비트 `integer` 제어변수 문제가 PR #5의 재작성 코드에도 남아있는지. 이 모듈을 통째로 다시 쓰는 PR이므로, **머지 시점에 C-1/G-9를 별도로 다시 적용할 필요가 없어질 가능성이 크다** — 반대로 PR #5가 이 문제를 새로 만들지 않았는지 확인 필요.
2. §D-1이 지적한 `mlkem_poly_accelerator`의 유일한 TB(`tb_mlkem_poly_accelerator.sv`)가 파이프라인 드레인/레이어 경계 케이스까지 커버하는지. PR #5 본문에는 새 TB 언급이 없다.
3. §G-13이 경고한 정확한 위험 — "동시 활성 DSP 수가 늘어 라우팅 혼잡과 DSP 입력 팬인이 변해 WNS +0.105ns 설계에서 타이밍이 악화될 가능성" — 에 대해 PR #5는 **이미 재합성까지 마쳤고 결과가 오히려 개선**되었다고 보고한다(WNS +0.150ns, +0.045ns 개선, 64세션 전체 블록디자인 기준). Vivado 버전이 2020.2(현 저장소 기준)가 아니라 **2025.2**로 보고되어 있어, 툴 버전 차이가 결과에 영향을 줬을 가능성을 실제 재현 시 감안해야 한다.
4. LUT/FF/DSP가 각각 24,081/25,727/51 → 22,232/26,882/59로 변하는데, **BRAM은 9 → 16 타일로 거의 2배 증가**한다(뱅크 분할의 직접적 대가). ZedBoard(XC7Z020) BRAM36 가용량 140 기준으로는 여전히 11.4%라 여유는 충분하지만, 다른 기능(64세션 BRAM 테이블 등)과 합산한 총 사용량 재확인이 필요하다.

**권장.**
1. PR #5는 `mergeable: true`이고 선행 리뷰의 최우선 성과 항목을 독립적으로, 더 나은 방식으로 달성했다고 보이므로 **가장 먼저 diff를 받아 rtl-engineer/verification-engineer가 직접 검토할 대상**으로 격상한다(선행 리뷰는 G-13을 "검증 완료된 설계를 건드리는 리스크"로 Medium/선택에 두었으나, 외부에서 이미 재합성까지 마친 구현이 존재한다는 사실이 그 리스크 평가를 낮춘다).
2. 병합 전 위 교차 검증 포인트 1~4를 확인한다. 특히 (2)의 파이프라인 드레인 TB 부재는 §H-2/D-1과 같은 성격의 위험(수정됐지만 전용 TB가 없는 로직)이므로, 병합과 동시에 `tb_mlkem_poly_accelerator.sv`에 레이어 경계·드레인 케이스를 추가해야 한다.
3. 이 프로젝트의 최종 목표는 2020.2 기준 재현이므로, PR #5를 가져온 뒤 **반드시 로컬 Vivado 2020.2에서 재합성**해 보고된 수치를 재확인한다(§G-0의 "기준선 없이는 어떤 변경도 평가할 수 없다" 원칙과 동일).

---

## C. [정정, 2026-09-18 재확인] G-1/G-3/G-4는 소실되지 않았다 — `external/`과 이 프로젝트의 커밋된 build tree를 혼동한 오판

**이 섹션의 최초 작성(2026-09-17)은 잘못된 결론이었다.** `31144f0`(`fix: apply High-priority RTL fixes...`)로 G-1/G-3/G-4가 실제 커밋되어 있는지 이 프로젝트 자체의 git 이력과 파일을 직접 재확인한 결과, 세 항목 모두 **정상 반영·커밋된 상태**임을 확인했다:

- G-4: `outputs/rtl_aead/rtl/aead_fixed64_engine.sv:154` — `if (tag_matches) begin tag_o <= poly_tag; ... end else begin ... tag_o <= '0; ...`로 게이팅됨. 주석("resubmitted with its own leaked tag to force acceptance")까지 포함해 반영 확인.
- G-3: `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:241-245` — "previously truncated silently and aliased onto slot 0" 주석과 함께 범위 검사 로직 확인.
- G-1: `outputs/rtl_aead/rtl/mlkem_poly_addsub_controller.sv:99-103` — `compress_1`이 비교기 2개(`message_bit=(poly_rdata_i>=16'd833)&&(poly_rdata_i<=16'd2496)`) 구현으로 교체되어 있음. G-1의 실제 대상은 (제목의 "tomsg_controller"라는 이름과 달리) 선행 리뷰의 "대상" 라인이 명시한 대로 `mlkem_poly_addsub_controller.sv`가 맞다 — 별도의 tomsg_controller.sv 파일은 이 저장소에 존재하지 않는다.

**어제(2026-09-17) 오판의 원인.** 이 재검토는 §H-3/H-4 재확인을 위해 `external/zedboard-pqc-secure`(대상 팀 저장소의 **읽기 전용 미러**, `.gitignore`로 버전관리 제외, 원칙상 매번 재-clone)를 다시 받아 그 안에서 `aead_fixed64_engine.sv:150`을 확인했다. `external/`은 애초에 팀 원격 저장소(`bobobomin/zedboard-pqc-secure`)의 사본이고, G-1/G-3/G-4는 **이 프로젝트 자체**의 `outputs/rtl_aead/`·`zed_pqc/` build tree(총 5벌, `31144f0`에 커밋됨)에 적용된 로컬 수정이다. 애초에 대상 팀 저장소에 업스트림하지 않은 로컬 수정이 그 팀 저장소의 미러에 없는 것은 **정상**이며, "소실"이 아니다. `external/`에 직접 write하지 않는다는 원칙(§ 권장 1, 여전히 유효)과, "G-1/G-3/G-4가 로컬에서 사라졌다"는 결론(오판)을 섞어 잘못 추론했다.

**정정된 권장.**
1. RTL 수정 작업은 이 프로젝트 자체의 build tree(`outputs/`, `zed_pqc/`)에서 하고 `external/`은 diff 생성용 참고 사본으로만 쓴다는 원칙은 유지한다.
2. G-1/G-3/G-4는 **재작업 불필요** — 이미 `31144f0`으로 커밋되어 있다. 2026-09-16 리뷰 파일의 "✅ 적용됨 (2026-09-17)" 표시는 정정할 필요 없이 그대로 유효하다.
3. 다만 G-1/G-2/G-9(§D-3 참고)가 재검증을 요구한 항목 — 특히 G-2(신규 tomsg/addsub compress_1 TB)와 G-9(제어 인덱스 폭 사이징) — 는 여전히 **미착수**이며 실제 미해결 상태다. 이는 "소실"이 아니라 애초에 손대지 않은 항목이므로 우선순위 표의 "재작업" 항목이 아니라 원래 리뷰의 우선순위 그대로 따른다.

---

## B-1. [2026-09-18 추가] PR #5 실제 diff 확보 — §B 교차 검증 포인트 1~4 결과

GitHub REST API로 PR #5의 실제 unified diff를 받았다(`GET /repos/bobobomin/zedboard-pqc-secure/pulls/5`, `Accept: application/vnd.github.v3.diff`). 5개 build-tree 사본에 동일하게 적용되어 있어(`31144f0`과 같은 패턴), 유일한 변경 원본인 `outputs/rtl_aead/rtl/*.sv`·`outputs/rtl_aead/tb/*.sv`만 라인 단위로 읽었다. Vivado/XSim이 이 환경에 없어 재합성·시뮬레이션은 **수행하지 못했고**, 아래는 정적 코드 대조만으로 확인 가능한 범위다.

**포인트 1 — C-1/G-9(32비트 `integer` 제어변수)가 재작성 코드에도 남아있는가: 남아있다, 미해결.**
`mlkem_poly_accelerator.sv`의 새 선언은 `integer layer, span, block_start, butterfly, zeta_index;`(`pair_index`만 `bm_ptr`이라는 이름으로 남고 여전히 `integer`는 아니지만 별도 폭 지정 없이 암묵적 폭). PR #5는 이 모듈을 파이프라인 구조로 통째로 재작성하면서도 G-9가 지적한 인덱스 폭 문제는 건드리지 않았다 — 새로 만든 `bf_d[1:5]`(8비트 `[7:0]`)·`bm_idx_d[1:8]`(7비트 `[6:0]`) 등 파이프라인 지연 레지스터는 오히려 올바르게 폭을 좁혀 선언했으면서, 기존 `integer` 제어변수는 그대로 두었다. **머지 후에도 G-9는 별도 항목으로 남는다.**

**포인트 2 — 파이프라인 드레인/레이어 경계 케이스를 커버하는 신규 TB가 있는가: 없다, 확인됨.**
`tb_mlkem_poly_accelerator.sv`의 diff는 새 `set_i` 포트에 맞춰 `set_sel` 신호를 배선하는 변경 4줄뿐이다(diff offset 1077-1117 부근). 레이어 경계에서 파이프라인이 실제로 드레인되는지, `BASEMUL_RUN`→`BASEMUL_DRAIN` 전환 시 8단 파이프라인에 채워진 값들이 올바르게 다 빠져나가는지를 겨냥한 새 테스트 케이스는 추가되지 않았다. 기존 TB가 256계수 전체를 수행하고 최종값만 비교하는 구조라면 레이어 경계를 우연히 통과는 하지만, **경계에서만 튀는 오프바이원류 버그는 최종값 검증만으로 놓치기 쉽다.** §D-1(H-2)이 지적한 "수정됐지만 전용 TB가 없는 로직"과 같은 성격의 위험이 PR #5에도 그대로 적용된다.

**포인트 3 — WNS 재현: 이 환경에서 수행 불가.** 로컬 Vivado 2020.2가 없어 보고된 수치(WNS +0.150ns)를 재확인하지 못했다. 이 항목은 미해결로 남는다.

**포인트 4 — BRAM 약 2배 증가: 코드로 구조적 확인됨.** `bank_a[0:255]` 1개(2포트)가 `bank_a0s0/a0s1/a1s0/a1s1[0:127]` 4개로, `bank_b[0:255]`/`bank_r[0:255]` 각 1개가 `bank_bs0/bs1`, `bank_rs0/bs1`(각 `[0:255]`, 즉 저장 용량 2배)로 바뀌었다. `set_i`로 아키텍처가 쓰는 절반과 호스트가 쓰는 절반을 나누는 더블버퍼링 구조이므로 보고된 BRAM 증가는 설계 의도와 일치한다.

**추가로 정적 검토에서 발견한, 선행 리뷰가 다루지 않은 새 리스크(PR #5 자체 도입분):**

- `mlkem_poly_bridge_controller.sv`가 순차(LOAD_A→LOAD_B→CORE→STORE) FSM에서 다음 명령의 로드와 이전 결과의 저장을 오버랩하는 FSM(`GO`/`OV_WAIT`/`FLIP`)으로 전면 재작성됐다. `ready_o` 산출식과 `GO`/`FLIP` 상태의 `set`/`stq`/`cmd`/`sd` 갱신 순서(비차단 대입이라 우변은 이전 사이클 값을 참조하는 점은 정상)는 읽어본 바로는 논리적으로 일관되나, **이 정도 복잡도의 오버랩 FSM은 코드 검토만으로 사이클 단위 정확성을 담보할 수 없다.** PR 본문이 언급한 대로 `tb_mlkem_poly_bridge_controller.sv`도 포트 목록에 `rdy` 추가뿐 실질적 새 테스트는 없다.
- `mlkem_shared_hash_engine.sv`도 메모리 페치를 `in_index`보다 앞서가는 `fetch_index` + 더블 버퍼(`word_cur`/`word_nxt`)로 바꿔 SHA3 피드를 매 사이클 1바이트로 유지하도록 재구성했다. 이 역시 신규 TB 없이 기존 TB만으로 커버된다.
- `tb_full_attack_fault_protection.sv`에서 `force`/`release` 대신 `fork...join` + 조건부 대입으로 fault 주입 방식을 바꿨다는 커밋 메모(XSim 최적화 버그 회피)가 있다 — 이 자체는 타당해 보이나, **왜 바꿨는지의 근거(XSim이 `force`가 걸린 `always_ff` 변수를 오컴파일한다)를 로컬에서 재현/검증할 수 없다.**

**결론 및 권장 갱신.** PR #5는 아키텍처 접근(뱅크 패리티 분할)이 타당하고 BaseMul/NTT/INTT를 실제로 파이프라인화했다는 점은 diff로 확인되지만, **파이프라인 드레인·오버랩 FSM 양쪽 다 이를 표적하는 신규 테스트가 없다.** §B 권장 2("병합과 동시에 `tb_mlkem_poly_accelerator.sv`에 레이어 경계·드레인 케이스를 추가")를 **병합의 전제조건**으로 격상할 것을 권한다 — 특히 이 프로젝트에도 로컬 Vivado/XSim이 없으므로, 실제 시뮬레이션 없이 병합하면 이 저장소의 §H-1/H-2와 같은 성격의 sim 미검증 구조적 리스크를 그대로 들여오는 셈이다.

---

## D. PR 5건이 해결하지 못한 문제 목록

PR #1~#5를 전부 반영해도(즉 #4/#5까지 머지된다고 가정해도) **여전히 남는 문제**를 선행 리뷰(2026-09-16) 기준으로 정리한다. PR #1~#3은 AXI 프론트엔드 버그 4건과 타이밍 클로저(Poly1305/ChaCha20/SHA3 경로 분할)만 다뤘고, #4는 그 타이밍 클로저의 중복 시도, #5는 ML-KEM 산술 파이프라이닝만 다룬다. 즉 **5건 모두 "동작 성능/타이밍" 아니면 "AXI 프로토콜 버그"** 범주이고, 아래 항목들은 그 범주 밖이라 어떤 PR도 건드리지 않는다.

### D-0. PR #1이 "고쳤다"고 주장하지만 실제로는 안 고쳐진 항목 (검증됨)

PR #1 본문: *"reset 방식 불일치: frontend 3개만 sync reset, 나머지 29개 모듈은 async였다... 전부 async assert로 통일했다."*

`main`의 실제 코드(`outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:148`)를 확인한 결과:

```systemverilog
always_ff @(posedge clk_i) begin
    if (!rst_ni) begin
```

`or negedge rst_ni`가 없다 — **여전히 동기 리셋**이다. 나머지 모듈(예: `poly1305_fixed96.sv:115`, `chacha20_block.sv:85`)은 `always_ff @(posedge clk_i or negedge rst_ni)`로 비동기 리셋이다. PR #1의 changelog와 실제 병합 결과가 어긋난다 — PR이 다른 프론트엔드 파일에는 적용했는데 이 파일을 놓쳤거나, 머지 과정에서 되돌아간 것으로 보인다. **선행 리뷰 §F-6이 지적한 항목과 동일하며, PR #1로 "해결됨"이라고 오인하기 쉬운 항목이므로 별도 표기.**

### D-1. 정확성/기능 버그 — 어떤 PR도 다루지 않음

| 항목 | 선행 리뷰 참조 | 내용 |
|---|---|---|
| 시뮬레이션 RTL ≠ 합성 RTL (`mlkem_poly_tomsg_controller`) | H-1 | XSim이 컴파일하는 사본과 비트스트림에 들어가는 사본이 `compress_1` 구현이 다름. 양방향 드리프트 |
| 위 모듈 테스트벤치 0개 | H-2 | 드리프트가 검증되지 않는 직접 원인 |
| HW 슬롯 범위 검사 부재 | H-3 | `slot_reg`에 64 이상 값을 쓰면 거부가 아니라 슬롯 0으로 앨리어싱. 방금 재확인 결과 **`main`에서 여전히 무방비**(§C 참고) |
| 인증 실패 시 태그 노출 (위조 오라클) | H-4 | `tag_o <= poly_tag;` 무조건 대입. **`main`에서 여전히 무방비** |
| Fault TB가 합성 안 되는 top을 때림 | H-5 | fail-closed 7종 fault 코드가 시뮬레이션에서 한 번도 직접 검증된 적 없음 |
| `mlkem_decaps_memory`의 `poly_mem` 주소가 배열 범위를 넘음 | C-6 | sim/synth 동작이 갈릴 수 있는 잠재 위험 |

### D-2. 보안 로직 — 어떤 PR도 다루지 않음

| 항목 | 참조 | 내용 |
|---|---|---|
| 세션 무효화/키 소거 경로 없음 | B-1 | `cfg_valid_i`가 상수 1로 묶여 있어 세션 회수 불가 |
| 설치 후 키 재료가 4중으로 잔존 (약 2,816 FF) | B-2 | 사용 후 소거 없음 |
| Fault 시 세션 카운터가 오염돼 RX 방향 영구 desync 가능 | B-3 | fail-closed 자체는 맞지만 카운터 갱신이 fault 신호를 안 봄 |
| session_id가 휘발성이라 PS 재부팅 후 키스트림 재사용 가능 | B-4 | 데모 플로우에서 재현 가능 |
| Fault 상태가 PS 드라이버에 노출되지 않고 STATUS 비트맵이 어긋남 | B-5 | `STATUS_KDF_BUSY`가 실제로는 fault 비트 |
| PS가 복호 중 ML-KEM 작업 메모리를 파괴할 수 있음 | B-6 | `REG_MEM_DATA` 쓰기가 `decap_busy_i`로 게이팅 안 됨 |

### D-3. RTL 구조/자원/타이밍 — PR #5가 우연히 건드릴 수 있는 것 vs 확실히 안 건드리는 것

- `mlkem_poly_accelerator.sv`를 통째로 재작성하는 PR #5가 병합되면 **C-1(32비트 integer 제어변수)은 자동으로 해소되거나 재작성판에서 재확인이 필요한 항목으로 바뀔 가능성이 높다** — PR #5 diff 확보 시 별도 재검토 필요(§B의 교차 검증 포인트 1과 동일).
- 그 외 **C-2(Keccak 1600비트 레지스터 중복), C-3(Poly1305/ChaCha20 레지스터 중복), C-5(watchdog이 case문에 덮어써짐), C-7(SLVERR 없음), C-8(비등록 read mux)은 PR #4/#5 어느 쪽 범위에도 없다.** PR #4가 SHA3 팬아웃은 다뤘지만(그리고 이미 main에 반영됨) Keccak 출력 레지스터 중복(C-2)은 별개 문제로, 손대지 않았다.

### D-4. 검증/저장소 위생 — 전부 미해결

D-1~D-5(TB 커버리지 공백, `$fatal` 미사용 TB 4개, 64세션 키분리 판정 약함, 드라이버 오류 버퍼 미클리어, MMIO 메모리 배리어 없음)와 F-1~F-7(4중 RTL 사본 비보호, closure 밖 파일 혼재, 절대경로 하드코딩, XDC 미포함, fault_inject 노출 등) 전부 5개 PR 중 어느 것도 언급하지 않는다.

### 결론

5개 PR을 전부 병합해도 프로젝트 README가 "보안 채널 가속기"로 내세우는 속성 중 **키 소거, 세션 회수, fault-카운터 정합성, HW 레벨 슬롯 방어, 인증 실패 시 정보 노출 차단** — 즉 보안 관련 항목 전부가 그대로 남는다. PR들은 순수하게 "더 빠르게" 또는 "AXI 프로토콜을 정확히" 만드는 데만 초점이 맞춰져 있다. 대회/심사 관점에서는 **PR #5(성능) 병합보다 §D-1/D-2(정확성·보안)를 먼저 처리하는 쪽이 리스크 대비 가치가 크다** — 특히 H-3/H-4는 수정 비용이 각각 게이트 1~2개 수준으로 매우 낮다.

---

## 요약 — 우선순위 [2026-09-18 갱신]

| 우선순위 | 항목 |
|---|---|
| **최우선 (변경 없음, diff 확보 완료)** | PR #5는 diff 확보·정적 검토(§B-1) 완료. C-1/G-9(제어변수 폭)는 재작성 후에도 미해결로 확인. **병합 전제조건**: `tb_mlkem_poly_accelerator.sv`(레이어 경계·드레인)와 `tb_mlkem_poly_bridge_controller.sv`(오버랩 FSM)에 신규 케이스 추가 + 로컬 Vivado 2020.2 재합성으로 WNS·기능 재확인 |
| ~~High (재작업)~~ **취소 — 이미 완료됨** | ~~선행 리뷰 G-1, G-3, G-4를 프로젝트에서 다시 적용~~ → 2026-09-18 재확인 결과 `31144f0`에 이미 커밋되어 있음(§C 정정 참고). "로컬 소실"은 `external/`(대상 팀 저장소 미러)과 이 프로젝트 자체 build tree를 혼동한 오판이었다 |
| **Low** | PR #4는 rebase 후 잔여 diff(AEAD 2,048비트 버스) 유무만 재확인하고, 없으면 close 권고 |
