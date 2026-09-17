# ZedBoard PQC 보안채널 가속기 코드 리뷰

- **리뷰 일자:** 2026-09-16
- **리뷰어:** `fpga-code-reviewer`
- **대상:** `zedboard-pqc-secure_with_AI` / branch `main` (read-only)
- **기준 상태:** 100 MHz, WNS +0.105 ns / TNS 0.000 ns, LUT 24,081 / FF 25,727 / BRAM 9 / DSP 51, 보드 검증 64/64 PASS

## 0. 리뷰 범위와 한계

### 라인 단위로 정독한 파일

| 구분 | 파일 |
|---|---|
| 통합/상위 | `mlkem_secure_channel_complete_axi_top.sv`, `mlkem_secure_channel_fault_protected_indexed_axi_top.sv`, `secure_channel_material_bram_core.sv` |
| 세션/AEAD | `aead_session_manager_bram.sv`, `aead_fixed64_engine.sv`, `poly1305_fixed96.sv`, `chacha20_block.sv` |
| 해시 | `keccak_f1600.sv`, `sha3_shake_stream.sv`, `mlkem_shared_hash_engine.sv` |
| ML-KEM 산술 | `mlkem_poly_accelerator.sv`, `mlkem_poly_addsub_controller.sv` |
| 프론트엔드/메모리 | `aead_traffic_indexed_axi_lite_frontend.sv`, `mlkem_decaps_axi_lite_frontend.sv`, `mlkem_decaps_memory.sv` |
| PS 드라이버 | `aead_hw.c/.h`, `mlkem_decaps_hw.c`, `secure_channel_hw.c`, `uart_secure_demo.c`, `pqc_64session_validation.c`, `pqc_session_scheduler.c` |
| IP 패키징 | `component.xml`, `validate_ip.tcl`, `system.bd`, `system_processing_system7_0_0.xci`, `system_xlconstant_0_0.xci` |
| TB | `tb_full_attack_fault_protection.sv`, `tb_mlkem_hash_g.sv` (전문), `tb_aead_fixed64.sv` / `tb_mlkem_secure_channel_complete_axi_top.sv` (부분) |

### 스킴(skim)만 한 파일 — 명시

아래 파일은 **모듈 인스턴스 그래프 추출 및 합성 closure 판정 목적으로만** 읽었고 라인 단위 검토는 하지 않았습니다. 재리뷰가 필요합니다.

- `mlkem512_decaps_aux_controllers.sv` (5개 모듈), `mlkem512_decaps_shared_engine.sv`, `mlkem512_kpke_decrypt_engine.sv`, `mlkem512_kpke_reencrypt_shared_engine.sv`, `mlkem512_unpack_controller.sv`, `mlkem512_codec_primitives.sv` (7개 모듈), `mlkem512_sampling_primitives.sv` (4개 모듈), `mlkem_poly_bridge_controller.sv`
- **합성 closure 밖 13개 변형 파일 전체** (§F-2 참고): `aead_axi_lite_wrapper.sv`, `aead_arbiter_4session.sv`, `aead_fixed64_wrapper.sv`, `aead_fixed64_decrypt_wrapper.sv`, `secure_channel_core.sv`, `secure_channel_material_core.sv`, `mlkem_session_kdf.sv`, `mlkem_hash_g.sv`, `mlkem_matrix_poly_generator.sv`, `mlkem_noise_poly_generator.sv`, `mlkem_handshake_transcript_hash.sv`, `mlkem512_decaps_engine.sv`, `mlkem512_kpke_reencrypt_engine.sv`, 대체 top 3종
- TB 31개 중 28개는 **DUT 인스턴스 목록과 `$error`/`$fatal` 밀도만** 집계했습니다.
- PS 미검토: `zed_pqc_bringup.c`, `software_crypto_benchmark.c`, `example_baremetal.c`, `zed_pqc_kat_vectors.h`, portable wrapper 2종
- PC 클라이언트 `uart_secure_client.py`는 세션 파생/AAD/논스 규약 정합성 확인 목적의 부분 검토만 했습니다.

### 자원/타이밍 판정의 전제

ZedBoard XC7Z020 기준 가용량은 LUT 53,200 / FF 106,400 / BRAM36 140 / DSP 220 입니다. 현재 사용률은 **LUT 45%, FF 24%, BRAM 6%, DSP 23%** 로, 이 설계에서 희소한 자원은 **자원이 아니라 타이밍(WNS +0.105 ns)** 입니다. 따라서 본 리뷰는 FF/LUT 절감 제안을 "여유 확보" 목적이 아니라 **라우팅 혼잡 완화를 통한 타이밍 여유 확보** 관점으로 우선순위를 매겼습니다. 조합 경로 깊이를 늘리는 제안은 별도로 위험 표기했습니다.

---

## A. 우선순위 High

### H-1. 시뮬레이션 RTL과 합성 RTL이 기능적으로 다름 (`mlkem_poly_addsub_controller.sv`)

**파일:**
- `outputs/rtl_aead/rtl/mlkem_poly_addsub_controller.sv:93-124`
- `zed_pqc/project_64session/project_1.srcs/sim_1/imports/src/mlkem_poly_addsub_controller.sv:49-69`

**문제.** 저장소에는 RTL 사본이 README가 경고한 2개가 아니라 **4개** 존재합니다.

| # | 경로 | 용도 |
|---|---|---|
| A | `outputs/rtl_aead/rtl/` | 독립 원본 |
| B | `zed_pqc/ip_repo/secure_channel_ip/src/` | 패키지 IP = **합성/비트스트림 소스** |
| C | `zed_pqc/project_64session/project_1.ipdefs/ip_repo/.../src/` | Vivado IP 정의 캐시 |
| D | `zed_pqc/project_64session/project_1.srcs/sim_1/imports/src/` | **XSim이 실제로 컴파일하는 사본** |

A = B = C 이지만(공백 차이만, §F-1), **D는 `mlkem_poly_tomsg_controller`가 기능적으로 다릅니다.**

합성되는 사본(A/B/C)의 `compress_1`:

```systemverilog
    integer index;logic[3:0]slot;logic[31:0]p;
    always_comb begin
        poly_addr_o=slot*256+index;
        ...
        p=poly_rdata_i*32'd1290168+32'h40000000;
    end
    ...
            WAIT:begin
                message_o[index]<=p[31];
```

시뮬레이션 사본(D)의 `compress_1`:

```systemverilog
    integer index;logic[3:0]slot;logic message_bit;
    /* Compress_1(x)=round(2x/q) mod 2 is 1 exactly on 833<=x<=2496 for
       canonical x in [0,3328].  Keep the two-cycle RAM schedule while
       replacing the old 16x32 constant multiply with two comparisons. */
    always_comb begin poly_addr_o=slot*256+index;busy_o=state!=IDLE;done_o=state==DONE;
        message_bit=(poly_rdata_i>=16'd833)&&(poly_rdata_i<=16'd2496);end
```

즉 **`16x32 상수 곱셈 → 비교기 2개` 최적화가 시뮬레이션 사본에만 적용되고 패키지 IP에는 반영되지 않았습니다.** 반대 방향의 드리프트도 있습니다: `mlkem_poly_addsub_controller`의 주소 생성은 A/B/C가 `poly_addr_o={sa,index}` (index를 `logic[7:0]`로 사이징)인데 D는 구버전 `poly_addr_o=sa*256+index` (`integer index`)입니다. **두 최적화가 서로 다른 사본에 하나씩만 반영되어 양방향으로 갈라진 상태입니다.**

두 `compress_1` 구현의 등가성은 `x ∈ [0,3328]` 범위에서만 성립합니다(경계값 832/833/2496/2497/3328에서 검증 완료 — 정상 범위에서는 동일). 그러나 Barrett reduction 이전의 비정규(non-canonical) 계수가 들어오면 곱셈판은 2^32 wrap, 비교판은 0을 반환하여 **결과가 갈립니다**. 지금은 정규 계수만 들어오므로 기능 오류는 없지만, **"XSim 기능 검증 통과"가 비트스트림 RTL에 대한 검증이 아니라는 사실 자체**가 문제입니다. OPTIMIZATION.md 절차 8번("XSim 기능 검증 → 구현 타이밍 → bitstream")의 전제가 현재 깨져 있습니다.

**개선 제안.**
1. `outputs/rtl_aead/rtl/mlkem_poly_addsub_controller.sv`의 `mlkem_poly_tomsg_controller`를 비교기 2개 버전으로 통일하고, A → B → 재패키징 → C 갱신 → D 재임포트 순서로 4개 사본을 정렬한다.
2. 자원/타이밍 효과: 16x32 상수 곱셈 1개 제거 → DSP 1개 또는 약 40~60 LUT 절감, `poly_rdata_i → p[31] → message_o` 조합 경로가 32비트 곱셈에서 16비트 비교기 2개 + AND 로 단축. **조합 깊이가 줄어드는 방향이므로 WNS에 유리**하나 정확한 값은 **재합성 없이 확인 불가 — 위험 항목으로 체크 필요**.
3. 사본 D는 별도 관리 대상에서 제외하고 Vivado sim_1 fileset이 B를 직접 참조하도록 변경하거나, CI에서 4개 사본의 SHA-256 일치를 검사한다.

**Priority: High**

---

### H-2. 기능이 갈라진 `mlkem_poly_tomsg_controller`에 테스트벤치가 전혀 없음

**파일:** `outputs/rtl_aead/tb/tb_mlkem_poly_addsub_streaming.sv:1-` (DUT: `mlkem_poly_addsub_controller`만)

**문제.** `mlkem_poly_addsub_controller.sv`는 **두 개의 모듈**을 담고 있습니다.

```
mlkem_poly_addsub_controller.sv:2:module mlkem_poly_addsub_controller(
mlkem_poly_addsub_controller.sv:93:module mlkem_poly_tomsg_controller(
```

TB는 앞의 것만 인스턴스화합니다. 즉 **H-1의 드리프트가 발생한 모듈은 단위 테스트가 0개**이며, 이것이 드리프트가 탐지되지 않은 직접적 원인입니다. `compress_1`은 복호 결과 메시지 비트를 만드는 함수이므로 여기서 틀리면 ML-KEM 복호가 조용히 실패합니다(그리고 Fujisaki-Okamoto 재암호화 비교에서 거부로 나타나 원인 추적이 어렵습니다).

**개선 제안.** `tb_mlkem_poly_tomsg_controller.sv`를 신규 작성하고, 최소한 다음 경계값을 전수 또는 표본 검증한다: `x = 0, 832, 833, 1664, 2496, 2497, 3328`. 256계수 전체를 0..3328 순회하며 소프트웨어 `compress_1` 참조값과 비교하는 exhaustive 테스트가 256 × 3329 사이클 수준이므로 부담 없이 가능합니다.

**Priority: High**

---

### H-3. 범위 밖 슬롯을 하드웨어가 구조적으로 거부할 수 없음

**파일:**
- `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:117-119`
- `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:241-242`
- `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv:78`
- `outputs/rtl_aead/ps_driver/pqc_64session_validation.c:141-145`

**문제.** 세션 매니저의 범위 검사는 항진명제입니다.

```systemverilog
    function automatic logic slot_in_range(input logic [SLOT_WIDTH-1:0] slot);
        slot_in_range = (slot < NUM_SESSIONS);
    endfunction
```

`SLOT_WIDTH = $clog2(64) = 6` 이므로 `slot`은 6비트이고 6비트 무부호값은 **항상** 64보다 작습니다. 합성 시 상수 1로 축약되며, `aead_session_manager_bram.sv:291`의 판정은 실질적으로 `!session_valid[req_slot_i]` 하나만 남습니다.

그리고 두 프론트엔드 모두 슬롯 레지스터에 **truncation**을 수행합니다.

```systemverilog
    REG_SLOT: slot_reg <= merge_wstrb(
        {{(32-SLOT_WIDTH){1'b0}}, slot_reg}, wdata_q, wstrb_q);
```

`merge_wstrb`는 32비트를 반환하는데 `slot_reg`는 `[5:0]`이므로 하위 6비트만 남습니다. ML-KEM 프론트엔드도 동일합니다.

```systemverilog
    REG_SLOT:if(wstrb_q[0])decap_slot_o<=wdata_q[SLOT_WIDTH-1:0];
```

따라서 **PS가 REG_SLOT에 64(0x40)를 쓰면 거부되는 것이 아니라 슬롯 0으로 앨리어싱됩니다.** 트래픽 경로에서는 슬롯 0의 키로 서비스되고, ML-KEM 경로에서는 **슬롯 0에 설치된 정상 세션의 키를 덮어씁니다.**

README "64세션 보드 검증 항목"의 *"범위 밖 슬롯 64 거부"* 는 하드웨어 검증 항목처럼 기술되어 있으나, 실제 테스트는 다음과 같습니다.

```c
    result = secure_channel_hw_encrypt(
        &device, 64u, probe, (uint8_t)(sizeof(probe) - 1u),
        ciphertext, tag, &counter);
    report("driver rejects out-of-range slot 64",
           result == AEAD_HW_ERR_ARGUMENT, &failures);
```

테스트 이름 자체가 `driver rejects`이고, `AEAD_HW_ERR_ARGUMENT`는 `aead_hw.c:167`의 C 레벨 `slot >= AEAD_HW_MAX_SESSIONS` 검사에서 반환됩니다. **AXI 트랜잭션은 발생조차 하지 않습니다.** 즉 거부는 100% PS 소프트웨어 방어이며 하드웨어 방어는 없습니다. 같은 AXI 주소를 쓰는 다른 PS 코드(또는 드라이버 수정/우회)는 앨리어싱을 그대로 유발합니다.

**개선 제안.**
1. 프론트엔드에서 `wdata_q`의 상위 비트를 검사하여 범위 초과 시 슬롯 레지스터를 갱신하지 않고 `s_axi_bresp <= RESP_SLVERR`를 반환하거나 sticky 오류 비트를 세운다. `aead_traffic_indexed_axi_lite_frontend.sv:241`에서 `if (|wdata_q[31:SLOT_WIDTH]) <error> else slot_reg <= ...` 형태.
2. 또는 슬롯 레지스터를 `[7:0]`로 넓혀 실제 범위 검사가 의미를 갖게 하고, `slot_in_range()`가 상수로 축약되지 않게 한다.
3. 자원 영향: 26비트 OR 리덕션 1개(약 6 LUT). 타이밍: `s_axi_wdata → slot_reg` 경로는 크리티컬 경로가 아니므로 위험 낮음. 다만 **재합성 없이 단정 불가**.
4. `slot_in_range()`는 현 파라미터에서 dead code임을 코멘트로 명시하거나 제거한다. 유지하려면 `NUM_SESSIONS`가 2의 거듭제곱이 아닌 경우를 위한 것임을 주석으로 남긴다.
5. README의 해당 항목을 "드라이버 레벨 거부"로 정정한다.

**Priority: High**

---

### H-4. 인증 실패한 암호문에 대해 계산된 Poly1305 태그가 외부로 노출됨

**파일:**
- `outputs/rtl_aead/rtl/aead_fixed64_engine.sv:148-163`
- `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:390`
- `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:141-142`

**문제.** 복호 경로에서 태그 비교 **전에** 계산된 태그를 출력 레지스터에 무조건 기록합니다.

```systemverilog
                S_WAIT_DECRYPT_TAG: begin
                    if (poly_done) begin
                        tag_o <= poly_tag;
                        if (tag_matches) begin
```

세션 매니저도 인증 결과와 무관하게 전달합니다.

```systemverilog
                        rsp_data_o    <= engine_auth_ok ? engine_data_out : 512'd0;
                        rsp_tag_o     <= engine_tag_out;
```

평문은 `engine_auth_ok`로 게이팅되지만 **태그는 게이팅되지 않습니다.** 그리고 프론트엔드가 이를 `REG_OUTPUT_TAG`(0x1c0~0x1cc)로 읽기 가능하게 노출합니다.

이는 **위조 오라클**입니다. 공격 흐름:
1. 임의 암호문 C와 쓰레기 태그를 현재 기대 RX 카운터 N으로 제출 → 인증 실패
2. 인증 실패 시 `aead_session_manager_bram.sv:392-406`에서 RX 카운터는 **증가하지 않으므로** 기대 카운터는 여전히 N
3. `REG_OUTPUT_TAG`에서 C에 대한 **정답 태그**를 읽음
4. 같은 C를 정답 태그와 함께 카운터 N으로 재제출 → `auth_ok = 1`

즉 Poly1305 인증이 무력화됩니다. 태그 비교 자체는 상수 시간으로 올바르게 구현되어 있는데(§E-1 참고) 출력 게이팅 누락이 그 이점을 상쇄합니다.

**현재 익스플로잇 경로 유무 — 정확한 판정.** 출하된 PS 앱(`uart_secure_demo.c:296-300`)은 인증 실패 시 `ERR AUTH`만 출력하고 `REG_OUTPUT_TAG`를 읽지 않으므로 **UART를 통한 실제 공격 경로는 현재 없습니다.** README 위협 모델에서 PS(기지국 제어부)는 신뢰 기반이므로 즉각적 침해는 아닙니다. 그러나 (a) AEAD 구현 규약("거부된 암호문에 대해 계산값을 절대 반환하지 않는다")의 무조건 위반이고, (b) 수정 비용이 1행이며, (c) 대회/심사에서 지적되기 쉬운 항목이므로 High로 둡니다.

**개선 제안.** 복호 방향에서 태그 출력을 인증 결과로 게이팅한다. 태그는 암호화 방향의 산출물이므로 복호 성공 시에도 굳이 반환할 필요가 없다.

- `aead_fixed64_engine.sv:150`: `tag_o <= poly_tag;` → `if (!decrypt_reg) tag_o <= poly_tag;` (또는 `tag_matches` 조건 추가)
- `aead_session_manager_bram.sv:390`: `rsp_tag_o <= engine_tag_out;` → `rsp_tag_o <= (engine_auth_ok && !active_decrypt_q) ? engine_tag_out : 128'd0;`

자원 영향: 128비트 AND 게이트 1단(약 32 LUT) 또는 write-enable 게이팅으로 0 LUT. 타이밍: `rsp_tag_o`는 레지스터 출력이고 이후 프론트엔드 레지스터로만 가므로 크리티컬 경로 아님. 위험 낮음.

**Priority: High**

---

### H-5. Fault/공격 테스트벤치가 합성되지 않는 top을 대상으로 함

**파일:** `outputs/rtl_aead/tb/tb_full_attack_fault_protection.sv:9`

**문제.** 20개의 `$fatal` 체크를 담은 유일한 공격/고장주입 TB가 다음을 DUT로 씁니다.

```systemverilog
  mlkem_secure_channel_fault_protected_axi_top dut(clk,rst,aw,awv,awr,wd,ws,wv,wr,bp,bv,br,
```

그러나 비트스트림에 들어가는 계층은 `component.xml:388`의 `modelName`이 지정한 `mlkem_secure_channel_complete_axi_top` → `mlkem_secure_channel_fault_protected_**indexed**_axi_top` → `secure_channel_material_**bram**_core` 입니다. TB가 때리는 `..._fault_protected_axi_top`은 `secure_channel_material_core`(레지스터 배열, 4세션)를 쓰는 **다른 변형이며 합성 closure에 포함되지 않습니다**(§F-2).

두 top의 차이는 사소하지 않습니다.
- 세션 수 4 vs 64, 슬롯 폭 2 vs 6
- 세션 저장소: 레지스터 배열 vs BRAM 24워드/슬롯 상태기계
- `outstanding` / `mon_rsp_valid` 응답 감시 로직(`..._indexed_axi_top.sv:204-207, 269-276`)

결과적으로 **출하 설계의 fail-closed 동작(F_SEQUENCE / F_HASH / F_SECRET / F_TRANSCRIPT / F_MATERIAL / F_TIMEOUT / F_OUTPUT 7종)은 시뮬레이션에서 한 번도 직접 검증되지 않았습니다.** 특히 `F_OUTPUT`는 §F-5에서 설명하듯 출하 비트스트림에서 **논리적으로 도달 불가**한 상태입니다.

추가로 이 TB에는 범위 밖 슬롯 테스트가 없고(H-3), 트래픽 프론트엔드의 슬롯 레지스터를 0 이외의 값으로 쓰는 케이스가 없습니다.

**개선 제안.**
1. `tb_full_attack_fault_protection.sv`를 `mlkem_secure_channel_complete_axi_top`(또는 최소한 `..._indexed_axi_top`) 대상으로 포팅한다. 기존 `request0`/`consume0` 태스크는 `aead_traffic_indexed_axi_lite_frontend`의 AXI 레지스터 접근(`aw(9'h00c, slot)` 등)으로 교체해야 한다. `tb_mlkem_secure_channel_complete_axi_top.sv`의 `aw`/`ar` 태스크를 재사용하면 작업량이 줄어든다.
2. 이식 후 7종 fault 코드 전수 + 다음 케이스를 추가한다: 슬롯 63 트래픽, 슬롯 64/127 쓰기 후 슬롯 0 키 보존 확인, fault 발생 중 트래픽 요청 발행(§B-3의 카운터 오염 확인), fault 후 재-launch 시 `fault_code_o` 클리어 확인.

**Priority: High**

---

## B. 우선순위 Medium — 보안 로직

### B-1. 세션 무효화 경로가 존재하지 않고 키 소거(zeroization)가 없음

**파일:**
- `outputs/rtl_aead/rtl/secure_channel_material_bram_core.sv:80`
- `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:261-277`
- `outputs/rtl_aead/ps_driver/uart_secure_demo.c:212`

**문제.** 세션 매니저에는 무효화 분기가 있습니다.

```systemverilog
                        end else if (!cfg_valid_i) begin
                            session_valid[cfg_slot_i] <= 1'b0;
                            cfg_done_o <= 1'b1;
```

그런데 상위에서 이 입력이 상수로 묶여 있습니다.

```systemverilog
        .cfg_valid_i(1'b1),
```

따라서 **출하 설계에는 세션을 회수할 수단이 전혀 없습니다.** `session_valid[slot]`은 설치되면 전역 리셋까지 1로 유지됩니다. 위 무효화 분기는 dead code로 합성에서 제거됩니다. 데모 코드가 이 사실을 알고 있습니다.

```c
    xil_printf("NOTE LEAVE is PS-logical; OPEN securely overwrites the slot\r\n");
```

즉 `LEAVE`는 PS RAM의 `session_active[slot]`만 0으로 만들고 PL은 키와 유효비트를 그대로 유지합니다. README "세션 leave/rejoin 반복 시험" 항목은 PS 소프트웨어 레벨 leave입니다.

키 소거에 대한 정확한 판정:
- **재설치 시 카운터 초기화는 실제로 동작합니다.** `S_CFG_WRITE`가 워드 0~23을 모두 쓰고 `config_word()`의 `else config_word = 32'd0`(`:138-139`)가 `WORD_TX_COUNT_LO`~`WORD_RESERVED`를 0으로 덮으므로, README의 "슬롯 재사용 시 counter 초기화" 주장은 RTL에서 확인됩니다.
- **그러나 전용 소거 경로는 없습니다.** 재설치로 덮어쓰기 전까지 이전 사용자의 32바이트 TX/RX 키가 BRAM에 남습니다. 그리고 `cfg_valid_i`가 1로 묶여 있어 "유효비트만 내리는" 부분 소거조차 불가능합니다.
- `session_valid`(64비트)는 설계의 나머지 부분이 쓰는 dual-rail(`ss_a == ~ss_b` 등) 이중화 보호를 받지 않는 단일 레지스터입니다. 고장주입으로 임의 비트가 1이 되면 소거되지 않은 잔존 키로 서비스가 이뤄집니다.

**개선 제안.**
1. `secure_channel_material_bram_core`에 `revoke_i` / `revoke_slot_i` 포트를 추가하고 `cfg_valid_i`에 연결한다. ML-KEM 프론트엔드에 `REG_REVOKE` 레지스터를 신설해 PS가 명시적으로 세션을 회수할 수 있게 한다.
2. 무효화 시 유효비트만 내리지 말고 **24워드 전체를 0으로 쓰는 소거 상태**를 추가한다. `S_CFG_WRITE` 상태기계를 재사용하고 `config_word()`가 `cfg_valid_q == 0`일 때 항상 0을 반환하게 하면 신규 상태 없이 구현 가능하다(24사이클).
3. `session_valid`에 이중화(`session_valid` / `~session_valid_b`)를 적용하고 불일치 시 `F_MATERIAL` 계열 fault를 올린다. 자원: FF 64개 + 64비트 비교기(약 20 LUT). **비교 트리가 새로 생기므로 타이밍 영향은 재합성 없이 확인 불가 — 위험 항목.** `..._indexed_axi_top.sv:242-244`처럼 비교 결과를 레지스터에 담고 다음 사이클에 판정하는 기존 패턴을 그대로 따르면 조합 깊이 증가를 피할 수 있다.

**Priority: Medium**

---

### B-2. 설치 후 키 재료가 4중으로 레지스터에 상주하며 소거되지 않음

**파일:**
- `outputs/rtl_aead/rtl/mlkem_shared_hash_engine.sv:112` (`digest_o`, 576비트)
- `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:322-323` (`material_a`/`material_b`, 1152비트)
- `outputs/rtl_aead/rtl/secure_channel_material_bram_core.sv:57` (`material_q`, 576비트)
- `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:288-289` (`ss_a`/`ss_b`, 512비트)

**문제.** 트래픽 키 재료(576비트 = TX키 32B + RX키 32B + 프리픽스 8B)가 설치 완료 후에도 네 곳의 레지스터에 그대로 남습니다.

```systemverilog
                K_WAIT: if (hdone_ctl) begin
                    material_a <= hdigest ^ {575'd0, fault_inject_i[3]};
                    material_b <= ~hdigest;
```

`REPORT → IDLE` 전이(`:339`)에서 이들을 지우는 코드가 없고, 다음 `launch`의 IDLE 분기(`:279-285`)도 `fault_detected_o` / `fault_code_o` / `hash_index` / `stage_timer`만 초기화합니다. `digest_o`는 다음 `start_i`(`:80`)에서만 클리어됩니다. ML-KEM 공유 비밀 `ss_a`/`ss_b`(512비트)도 동일합니다.

합계 **약 2,816 FF(576 + 1152 + 576 + 512)가 마지막 세션의 키 재료와 공유 비밀을 무기한 보유**합니다. 전체 FF 25,727의 11%입니다. 하드웨어 보안 가속기에서 사용 후 중간 키 소거는 기본 요구사항이며, JTAG readback 또는 고장주입으로 이 값들이 회수되면 마지막 세션 전체가 복호됩니다.

**개선 제안.** `REPORT` 상태에서 소거한다. 이중화 불변식(`x_a == ~x_b`)을 깨지 않도록 짝을 맞춰 0 / 전부1로 리셋한다.

```systemverilog
                REPORT: begin
                    ss_a <= 0;           ss_b <= ~256'd0;
                    transcript_a <= 0;   transcript_b <= ~256'd0;
                    material_a <= 0;     material_b <= ~576'd0;
                    state <= IDLE;
                end
```

`mlkem_shared_hash_engine`은 `DONE` 상태(`:124`)에서 `digest_o <= 0`을 추가한다. 단 `C_KDF` 결과는 상위가 `K_WAIT`에서 이미 캡처한 뒤이므로 안전하다. `secure_channel_material_bram_core`는 `accepted && cfg_done`(`:65-68`)에서 `material_q <= '0'`을 추가한다.

**자원/타이밍.** 신규 로직 없음. 해당 레지스터들은 이미 리셋 경로(상수 0 / 전부1 로드)를 가지고 있으므로 `REPORT` 상태를 인에이블 조건에 OR로 추가하는 것에 그친다. **LUT/FF 증가 없음, 조합 깊이 증가 없음.** 비용 대비 효과가 가장 좋은 보안 개선 항목입니다.

**Priority: Medium**

---

### B-3. Fault 발생 시 세션 카운터가 오염되어 RX 방향이 영구 desync될 수 있음

**파일:**
- `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:204-211`
- `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:392-417`

**문제.** fail-closed 자체는 올바르게 구현되어 있습니다. 검증 결과:

| 출력 | fault 시 동작 | 판정 |
|---|---|---|
| `rsp_auth_ok_o` | `raw_rsp_auth & outstanding & !fault_detected_o` (`:206`) | de-assert ✓ |
| `rsp_data_o` | `fault_detected_o ? 512'd0 : raw_rsp_data` (`:211`) | 0으로 마스킹 ✓ |
| `rsp_error_o` | `raw_rsp_error \| (fault_detected_o & mon_rsp_valid)` (`:207`) | assert ✓ |
| `install_req` | `(state==C_START) && !fault_detected_o && ...` (`:201-202`) | 키 설치 차단 ✓ |
| `final_fail` | `core_fail \| fault_detected_o` (`:219`) | PS에 fail 보고 ✓ |

**문제는 세션 매니저가 `fault_detected_o`를 전혀 보지 않는다는 점입니다.** 매니저는 자체 판단으로 `engine_auth_ok`가 1이면 BRAM 카운터를 증가시킵니다.

```systemverilog
                        if (engine_auth_ok) begin
                            if (active_decrypt_q) begin
                                updated_counter_q <= active_rx_counter + 64'd1;
```

상위가 응답을 0으로 마스킹하고 오류로 보고해도 **BRAM의 RX 카운터는 이미 N+1로 올라갑니다.** PS와 PC는 해당 패킷이 거부된 것으로 알고 카운터 N을 유지하므로, 이후 모든 수신 패킷이 `S_CHECK`의 `request_counter_q != active_rx_counter`(`:357`)에서 거부됩니다. **재싱크 수단이 없어 리셋 전까지 해당 슬롯의 수신 방향이 영구 불능**입니다. 송신 방향도 동일하게 TX 카운터만 소모됩니다.

**개선 제안.**
1. `secure_channel_material_bram_core`에 `fault_i` 입력을 추가해 상위의 `fault_detected_o`를 세션 매니저까지 전달하고, `S_ENGINE_WAIT`에서 `engine_auth_ok && !fault_i` 조건으로 카운터 갱신을 게이팅한다. 자원: AND 게이트 1개. 타이밍 위험 없음.
2. 또는 PS 측에 재싱크 절차를 제공한다(`REG_RSP_COUNTER_LO/HI`가 이미 기대 카운터를 반환하므로 PC가 이를 신뢰해 자기 카운터를 맞추는 방식). 다만 이 경로는 공격자가 카운터를 임의로 전진시킬 수 있게 되므로 1번을 권장한다.

**Priority: Medium**

---

### B-4. 세션 ID 생성이 휘발성 카운터에 의존하여 PS 재부팅 후 키스트림 재사용 가능

**파일:** `outputs/rtl_aead/ps_driver/uart_secure_demo.c:134-137, 189-201, 243-247`

**문제.** 먼저 **설계가 올바른 부분**을 확인했습니다. 트래픽 키는 `material = SHAKE256("ZYNQ-PQCC-v1" || ss || transcript, 72)`이고, `transcript = SHA3(pk || ct || session_id)` 입니다. `mlkem_shared_hash_engine.sv:94-95`에서 transcript 입력의 마지막 4바이트가 `sid`이므로 **session_id가 키 파생에 확실히 반영됩니다.** PC 측(`uart_secure_client.py:34-36`)도 `public_key + kem_ciphertext + session_id.to_bytes(4,"big")`로 동일하게 계산합니다. 따라서 "64슬롯이 같은 KAT 암호문을 쓰는데 키가 같아지지 않는가"라는 우려는 **해당되지 않습니다.**

그러나 session_id 생성이 휘발성입니다.

```c
static uint32_t make_session_id(uint8_t slot, uint32_t generation)
{
    return 0x60000000u | ((generation & 0x003fffffu) << 6) | slot;
}
```

`session_generation[]`은 `uart_secure_demo_run`의 스택 배열이며 `memset(session_generation, 0, ...)`(`:200`)로 매 실행마다 0으로 초기화됩니다. 데모가 고정 KAT 암호문 `zed_kat_kem_ciphertext`(`:148`)를 쓰므로, **PS 재부팅 후 슬롯 N의 첫 `OPEN`은 이전 부팅과 완전히 동일한 session_id → 동일한 transcript → 동일한 TX/RX 키 + 프리픽스를 만들고, 카운터는 0으로 리셋됩니다.** ChaCha20 논스는 `prefix || counter_be`이므로 **키와 논스가 동시에 재사용되어 키스트림이 반복**됩니다. 두 부팅 세션의 같은 카운터 패킷을 XOR하면 평문 XOR이 노출됩니다.

실제 배포에서는 클라이언트가 매번 새 ML-KEM 암호문을 제시하므로 ss가 달라져 발생하지 않습니다. 그러나 **출하된 데모/검증 플로우에서는 재현 가능**하며, 키 유일성이 "PS가 session_id를 재사용하지 않는다"는 소프트웨어 불변식에만 의존하는 구조 자체가 취약합니다.

**개선 제안.**
1. 단기: `session_generation`을 비휘발 저장(QSPI) 또는 `XTime_GetTime()` 기반 초기값으로 시드하고, 데모 배너에 "고정 KAT 암호문 사용 — 키 유일성은 session_id에만 의존" 경고를 추가한다.
2. 근본: PL 측에 슬롯별 단조 증가 install-counter를 두고 KDF 입력에 포함시켜, PS가 무엇을 쓰든 하드웨어가 키 유일성을 보장하게 한다. `mlkem_shared_hash_engine`의 `C_KDF` 입력 길이를 75 → 79로 늘리고 `:96-98` 분기에 4바이트를 추가하면 된다(PC 클라이언트 동시 수정 필요). 자원: FF 64×64슬롯은 과하므로 전역 카운터 1개(64 FF) + 슬롯 바인딩으로 충분하다.
3. `pqc_64session_validation.c:141-145`와 별도로, 같은 session_id로 두 번 `OPEN`했을 때 키스트림이 반복되는지 확인하는 네거티브 테스트를 추가한다.

**Priority: Medium**

---

### B-5. Fault 상태가 PS 드라이버에 전혀 노출되지 않고 STATUS 비트 맵이 어긋남

**파일:**
- `outputs/rtl_aead/ps_driver/aead_hw.c:11-35`
- `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:114-129`

**문제.** 드라이버의 STATUS 비트 정의와 출하 RTL이 다릅니다.

드라이버:
```c
#define STATUS_CFG_PENDING   (1u << 4)
#define STATUS_KDF_BUSY      (1u << 6)
```

RTL:
```systemverilog
            REG_STATUS: begin
                read_mux[0] = inflight;
                read_mux[1] = done_q;
                read_mux[2] = auth_q;
                read_mux[3] = error_q;
                read_mux[4] = 1'b0;
                read_mux[5] = pending;
                read_mux[6] = fault_detected_i;
            end
```

- 비트 4는 RTL에서 **상수 0**인데 드라이버는 `STATUS_CFG_PENDING`으로 `wait_request_idle`(`aead_hw.c:89-91`)의 clear_mask에 넣습니다. 항상 0이므로 무해하지만 의미 없는 검사입니다.
- **비트 6은 RTL에서 `fault_detected`인데 드라이버는 `STATUS_KDF_BUSY`로 명명했습니다.** 이름대로 사용하는 코드가 추가되면 fault를 "KDF 사용 중"으로 오독합니다.
- `REG_FAULT`(0x024, `fault_code` + `fault_detected`)를 읽는 코드가 `ps_driver/` 전체에 **한 줄도 없습니다**(grep 확인). 따라서 dual-rail 비교, 해시 스케줄 감시, watchdog으로 구성된 7종 fault 서브시스템의 결과가 소프트웨어에 전달되지 않습니다.
- 결과적으로 `aead_hw_decrypt`는 "태그 불일치"와 "하드웨어 고장 검출"을 모두 `AEAD_HW_ERR_AUTH`로 반환합니다(`:222-223`). **고장주입 공격이 평범한 불량 패킷과 구분되지 않아 로깅/에스컬레이션이 불가능**합니다.
- 추가로 `REG_CFG_SESSION_ID 0x028u`(`:11`)는 출하 RTL의 읽기 전용 `REG_RSP_SLOT = 9'h028`과 주소가 충돌합니다. 현재 사용되지 않지만 잠재 함정입니다.

이 상수들(`REG_TX_KEY_BASE`, `REG_KDF_CONTROL`, `REG_SHARED_SECRET` 등 `:13-23`)은 합성되지 않는 `aead_axi_lite_wrapper` 계열의 레지스터 맵으로 보이며, 드라이버 헤더가 다른 프론트엔드 기준으로 남아 있습니다.

**개선 제안.**
1. `STATUS_KDF_BUSY` → `STATUS_FAULT`로 개명하고, `AEAD_HW_ERR_FAULT` 반환 코드를 신설한다. `aead_hw_encrypt`/`aead_hw_decrypt`가 `status & STATUS_FAULT`를 우선 검사해 `AEAD_HW_ERR_FAULT`를 반환하도록 한다.
2. `int aead_hw_read_fault(const aead_hw_t*, uint8_t *code)`를 추가해 `REG_FAULT`(0x024)의 `fault_code`를 노출하고, `uart_secure_demo.c`가 `ERR AUTH` 대신 `ERR FAULT <code>`를 출력하도록 한다. `fault_code_o`는 다음 `launch`에서 클리어되므로(`..._indexed_axi_top.sv:282`) **오류 직후에 읽어야 함**을 주석으로 명시한다.
3. 사용되지 않는 `REG_CFG_*` / `REG_TX_KEY_*` / `REG_KDF_*` / `REG_SHARED_SECRET` / `REG_TRANSCRIPT_HASH` 상수와 스텁 3종(`aead_hw_configure_session`, `aead_hw_clear_session`, `aead_hw_install_mlkem_session` — 전부 `ERR_UNSUPPORTED` 반환)을 제거하거나, 헤더 주석(`aead_hw.h:42-47`)이 동작하는 기능처럼 서술한 부분을 정정한다.

**Priority: Medium**

---

### B-6. PS가 복호 진행 중 ML-KEM 작업 메모리를 파괴할 수 있음

**파일:**
- `outputs/rtl_aead/rtl/mlkem_decaps_memory.sv:29-53`
- `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv:87-91`
- `outputs/rtl_aead/ps_driver/mlkem_decaps_hw.c` (`load`, `mlkem_decaps_hw_start`)

**문제.** `mlkem_decaps_memory`의 각 뱅크는 호스트 포트와 엔진 포트가 **동일 BRAM에 독립적으로 쓰기**를 수행합니다.

```systemverilog
    always_ff @(posedge clk_i) begin
        if(host_valid_i&&host_we_i&&host_region_i==0)sk_mem[host_addr_i[8:0]]<=host_wdata_i;
        host_sk_q<=sk_mem[host_addr_i[8:0]];
    end
    always_ff @(posedge clk_i) begin
        if(sk_we_i)sk_mem[sk_addr_i]<=sk_wdata_i;
        sk_rdata_o<=sk_mem[sk_addr_i];
    end
```

호스트 쓰기는 `decap_busy_i`로 게이팅되지 않습니다. 프론트엔드의 `REG_MEM_DATA` 처리부는 진행 상태를 보지 않습니다.

```systemverilog
                    REG_MEM_DATA:begin
                        if(wstrb_q==4'hF)begin mem_access_addr<=mem_addr;mem_valid<=1;
                            mem_we<=1;mem_addr<=mem_addr+12'd1;end
```

`decap_start_o`만 `!decap_busy_i`로 보호됩니다(`:76`). 따라서 **복호가 진행되는 1,052 µs 동안 PS가 `REG_MEM_DATA`에 쓰면 엔진의 `sk_we_i`/`ct_we_i`/`poly_we_i` 쓰기와 동일 주소에서 충돌**합니다. True dual-port BRAM의 동일 주소 동시 쓰기는 정의되지 않은 동작입니다. `poly` 영역은 복호 전 구간에 걸쳐 엔진이 집중적으로 쓰므로 위험이 가장 큽니다.

드라이버도 방어하지 않습니다. `mlkem_decaps_hw_start`는 CT를 `load()`한 뒤에야 제어 레지스터를 쓰며, busy 폴링을 하지 않습니다. 순차 사용(`secure_channel_hw_establish_session`이 `start` → `wait`) 시에는 안전하지만 `mlkem_decaps_hw_start`는 busy 검사가 없는 공개 API입니다.

**개선 제안.**
1. RTL: `REG_MEM_DATA` 쓰기/읽기를 `!decap_busy_i` 조건으로 게이팅하고, busy 중 접근에는 `s_axi_bresp <= RESP_SLVERR`를 반환한다. 자원: AND 게이트 1개. 타이밍 위험 없음. 가장 확실한 방어.
2. 드라이버: `mlkem_decaps_hw_start` 진입부에 STATUS 비트 1(`decap_busy`) 폴링을 추가한다.

**Priority: Medium**

---

## C. 우선순위 Medium — RTL 구조 / 자원 / 타이밍

### C-1. `mlkem_poly_accelerator`의 제어 인덱스가 전부 32비트 `integer`

**파일:** `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:44`

**문제.**
```systemverilog
    integer layer, span, block_start, butterfly, zeta_index, pair_index;
```

6개 제어 변수가 모두 32비트 부호 있는 `integer`입니다. 실제 필요 폭은 `layer` 3비트, `span`/`block_start`/`butterfly` 9비트, `zeta_index` 8비트, `pair_index` 7비트입니다. 이 변수들이 만드는 32비트 연산:

```systemverilog
                NTT_WRITE: begin
                    if (butterfly == block_start+span-1) begin
                        if (block_start+2*span >= 256) begin
```
```systemverilog
                            else begin layer<=layer-1; span<=span<<1; block_start<=0;
                                butterfly<=0; zeta_index<=(1<<(layer-1))-1;state<=INTT_READ; end
```

- `butterfly == block_start+span-1`: 32비트 가산기 + 32비트 비교기. NTT/INTT 각 1개씩.
- `block_start+2*span >= 256`: 32비트 가산기 + 32비트 비교기. NTT/INTT 각 1개씩.
- `zeta_index<=(1<<(layer-1))-1`: **32비트 가변 배럴 시프터**. `layer ∈ {2..7}`만 사용하므로 6엔트리 LUT로 충분합니다.
- `a_addr1=butterfly+span`, `a_addr0=2*pary_index` 등 주소 생성에서도 32비트 연산 후 8비트 절단.

**정량 추정.** FF는 6×32 = 192개 대비 사이징 시 약 44개 → **약 148 FF 절감**. LUT는 32비트 가산기 4개 + 32비트 비교기 4개(약 250 LUT) 대비 9비트 버전(약 70 LUT) → **약 180 LUT 절감**, 여기에 배럴 시프터 제거분이 더해집니다.

**타이밍 관점이 더 중요합니다.** `butterfly == block_start+span-1`은 32비트 캐리 체인이 FSM 다음 상태 로직으로 직결되는 조합 경로입니다. 9비트로 줄이면 캐리 체인 길이가 1/3.5로 단축됩니다. **조합 깊이가 감소하는 방향이므로 WNS에 유리**하지만, 정확한 개선폭은 **재합성 없이 확인 불가 — 위험 항목으로 체크 필요.** OPTIMIZATION.md는 곱셈/Barrett 경로 분할만 다루고 제어 경로 폭은 다루지 않으므로 중복 제안이 아닙니다. 참고로 같은 파일의 `mlkem_shared_hash_engine.sv:23-26`에는 이미 정확히 이 교훈을 적용한 주석이 있습니다("feeding these as 32-bit integers builds a 32-bit decoder"). 동일 원칙을 이 파일에 적용하지 않은 누락입니다.

**개선 제안.**
```systemverilog
    logic [2:0] layer;
    logic [8:0] span, block_start, butterfly;
    logic [7:0] zeta_index;
    logic [6:0] pair_index;
```
`zeta_index<=(1<<(layer-1))-1`은 `case(layer) 7:8'd63; 6:8'd31; ... endcase` 또는 `(8'd1 << (layer-1)) - 8'd1`로 폭을 고정한다. `span<=span>>1` / `span<=span<<1`은 9비트 내에서 안전하다(span 최대 128).

**주의.** 폭 축소 시 `block_start+2*span >= 256` 비교가 9비트에서 오버플로우할 수 있다(`block_start` 최대 254, `2*span` 최대 256 → 최대 510, 9비트 최대 511). 경계이므로 10비트로 잡거나 비교식을 `block_start + 2*span != 256`(항상 정확히 256에 도달) 로 바꾸는 것이 안전하다. **이 경계 조건 때문에 변경 후 `tb_mlkem_poly_accelerator.sv`의 NTT/INTT/BaseMul 3개 커맨드 전수 재검증이 필수**이다.

**Priority: Medium**

---

### C-2. `keccak_f1600`의 1600비트 출력 레지스터가 중복

**파일:**
- `outputs/rtl_aead/rtl/keccak_f1600.sv:12, 105-129`
- `outputs/rtl_aead/rtl/sha3_shake_stream.sv:135-141, 167-170`

**문제.** Keccak은 1600비트 상태를 두 벌 유지합니다.

```systemverilog
    logic [1599:0] state_reg;
    logic [1599:0] next_state;
```
```systemverilog
            end else if (busy_o) begin
                if (round_index == 5'd23) begin
                    state_o <= next_state;
                    busy_o  <= 1'b0;
                    done_o  <= 1'b1;
                end else begin
                    state_reg   <= next_state;
```

마지막 라운드만 `state_o`로 가고 나머지는 `state_reg`로 갑니다. `state_o`는 출력 포트이므로 1600 FF이 추가로 소비됩니다. 유일한 소비자인 `sha3_shake_stream`은 `permutation_done`이 뜨는 사이클에 즉시 복사합니다.

```systemverilog
                S_PERM_ABSORB: if (permutation_done) begin
                    sponge_state <= permutation_output;
```

`state_reg <= next_state`를 마지막 라운드에도 수행하고 `assign state_o = state_reg`로 바꾸면 **1,600 FF(전체 FF의 6.2%)를 제거**할 수 있습니다. 동작은 동일합니다: 마지막 라운드 클록 에지에 `state_reg`가 결과를 받고 같은 에지에 `done_o`가 1이 되므로, 소비자가 보는 타이밍이 변하지 않습니다. 조합 경로 길이도 동일합니다(`next_state → 레지스터`가 유지되고 `state_o`가 와이어가 될 뿐).

Keccak 서브시스템 전체로 보면 `state_reg`(1600) + `state_o`(1600) + `sponge_state`(1600) = 4,800 FF에서 3,200 FF로 줄어듭니다.

**주의 — 안전성 조건.** 이 변경은 "소비자가 `done_o` 어서션 사이클에 `state_o`를 캡처하고, 다음 `start_i`까지 최소 1사이클 간격이 있다"는 전제에서만 안전합니다. 현재 `sha3_shake_stream`은 이를 만족합니다(`S_PERM_*` → `S_ABSORB`/`S_SQUEEZE` 전이 후에야 `permutation_start`가 다시 뜸). `keccak_f1600`의 유일한 인스턴스는 `sha3_shake_stream.sv:77`이므로 현재는 문제없으나, **이 제약을 모듈 헤더 주석으로 명시**해야 다른 곳에서 재사용될 때 사고를 막을 수 있습니다.

**FF 절감 자체의 가치에 대한 솔직한 평가.** FF 사용률이 24%에 불과하므로 FF 확보가 목적은 아닙니다. 가치는 **1600비트 레지스터 뱅크 하나와 그에 딸린 라우팅이 사라져 혼잡이 완화되고, Keccak 라운드 경로(현재 WNS 크리티컬 경로의 최유력 후보)의 배치 자유도가 올라가는 것**입니다. 실제 WNS 개선은 **재합성 없이 확인 불가 — 위험/기회 항목으로 체크 필요.**

**Priority: Medium**

---

### C-3. Poly1305 / ChaCha20의 메시지·상태 레지스터 중복

**파일:**
- `outputs/rtl_aead/rtl/poly1305_fixed96.sv:37, 151`
- `outputs/rtl_aead/rtl/aead_fixed64_engine.sv:41, 114-116, 130-132`
- `outputs/rtl_aead/rtl/chacha20_block.sv:15-16, 99-118`

**문제 (a) — Poly1305 메시지 이중 보관.** 엔진이 768비트 메시지를 구성해 포트로 넘기고, Poly1305가 이를 다시 자기 레지스터로 복사합니다.

```systemverilog
                            poly_message_reg[127:0]   <= aad_reg;
                            poly_message_reg[639:128] <= data_reg;
                            poly_message_reg[767:640] <= {64'd64, 64'd16};
```
```systemverilog
                        message_reg <= message_i;
```

`poly_message_reg`는 Poly1305 동작 전 구간에 안정적이므로 `message_reg`는 순전히 중복입니다. `current_block = message_reg[128*block_index +: 128]`(`:76`)을 `message_i`로 바꾸면 **768 FF 절감**, 경로 길이는 불변(부모 레지스터 Q에서 출발하는 것은 동일).

추가로 `poly_message_reg[767:640] <= {64'd64, 64'd16}`는 **128 FF이 상수를 저장**하고 있습니다. `current_block`의 `block_index==5` 케이스에서 상수를 직접 선택하면 제거됩니다.

**문제 (b) — ChaCha20 `initial_state` 중복.** feed-forward용 초기 상태 512비트를 별도 레지스터로 보관합니다.

```systemverilog
                initial_state[0] <= 32'h61707865;
                ...
                for (i_seq = 0; i_seq < 8; i_seq = i_seq + 1) begin
                    initial_state[4+i_seq] <= key_i[32*i_seq +: 32];
```

그런데 word 0~3은 상수, word 4~11은 부모의 `key_reg`, word 12는 `chacha_counter`, word 13~15는 부모의 `nonce_reg`로, **512비트 전부가 이미 다른 곳에 존재하는 값의 복사본**입니다. `initial_state`를 `always_comb`로 재구성하면 **512 FF 절감**. feed-forward 덧셈(`:126`)의 입력이 레지스터 Q에서 오는 것은 동일하므로 조합 깊이 불변.

**합계 약 1,408 FF(768 + 128 + 512), 전체 FF의 5.5%.** (a)와 (b) 모두 로직을 **제거**하는 변경이므로 타이밍 리스크가 낮으나, **재합성 없이 단정 불가**.

**주의.** (b)는 ChaCha20의 안전성이 "부모가 `key_reg`/`nonce_reg`/`chacha_counter`를 동작 전 구간 안정적으로 유지한다"에 의존하게 됩니다. `aead_fixed64_engine`은 이를 만족하지만(`S_IDLE`에서만 갱신), `chacha_counter`는 `S_WAIT_POLY_KEY`에서 0→1로 바뀝니다(`:120`). 이때 `chacha_start`도 같이 뜨므로 새 블록의 시작과 동시이며 문제없으나, **모듈 헤더에 "입력은 `busy_o` 동안 안정 유지 필요" 계약을 명시**해야 합니다. 명시 없이 변경하면 재사용 시 위험합니다.

**Priority: Medium**

---

### C-4. BaseMul / NTT / INTT 루프가 파이프라인되지 않아 곱셈기가 대부분 유휴

**파일:** `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:296-348` (BaseMul), `:213-240` (NTT), `:262-294` (INTT)

**문제.** BaseMul은 계수쌍 1개당 9개 상태를 순차 실행합니다.

```
BASEMUL_READ → LATCH → MUL → MONT → REDUCE → ZMUL → ZMONT → ZREDUCE → WRITE
```

128쌍 × 9 = 1,152 사이클로, OPTIMIZATION.md가 보고한 1,154 사이클과 일치합니다. 문제는 `BASEMUL_MUL`의 4개 곱셈기, `BASEMUL_MONT`의 4개, `BASEMUL_REDUCE`의 4개가 **각각 9사이클 중 1사이클만 일하고 8사이클을 유휴**로 보낸다는 점입니다. 동일하게 NTT는 5상태/버터플라이, INTT는 7상태/버터플라이입니다.

**OPTIMIZATION.md와의 중복 여부.** 문서는 "BaseMul 조합 경로를 레지스터로 분할해 1,026 → 1,154 사이클(+128)"이라는 **처리량-주파수 절충**을 기록했습니다. 여기서 제안하는 것은 반대 축입니다: 각 스테이지의 곱셈 1개 구조를 **그대로 유지한 채** 스테이지를 겹쳐 실행하는 소프트웨어 파이프라이닝입니다. 문서에 없는 제안이며, **조합 깊이를 전혀 늘리지 않습니다**(각 스테이지는 여전히 곱셈 1개).

**의존성 분석 — 합법성 확인.**
- NTT: 레이어 내에서 버터플라이 `i`는 `a[i]`, `a[i+span]`만 읽고 쓴다. 같은 블록 내 서로 다른 `i`는 서로 다른 주소를 건드리므로 레이어 내 RAW 해저드가 없다. 레이어 경계에서는 해저드가 있으므로 파이프라인을 드레인해야 한다(7회 × 스테이지 수 = 약 35사이클, 무시 가능).
- `span >= 2`가 항상 성립하므로(NTT: 128→2, INTT: 2→128) `butterfly`와 `butterfly+span`이 절대 같은 주소가 되지 않는다. 현재 `NTT_WRITE`의 dual-port 동시 쓰기가 안전한 것도 이 덕분이다.
- BaseMul: 쌍 간 의존성이 전혀 없고, 읽기는 `bank_a`/`bank_b`, 쓰기는 `bank_r`로 뱅크가 분리되어 있다.

**제약 — BRAM 포트 수가 한계.** 완전 파이프라인(1쌍/사이클)은 `bank_a`에 매 사이클 2읽기 + 2쓰기 = 4포트를 요구하므로 불가능합니다(BRAM은 2포트). 현실적 목표는 **2사이클/연산**입니다: 사이클 0에 2읽기, 사이클 1에 4스테이지 전의 결과 2쓰기. 2사이클 동안 포트당 읽기 1회 + 쓰기 1회로 2포트 내에 수용됩니다.

**정량 추정 (2사이클/연산 기준).**

| 구간 | 현재 | 예상 | 절감 |
|---|---:|---:|---:|
| NTT TOTAL (5상태 → 2) | 17,928 | 약 7,200 | 약 10,700 |
| INTT TOTAL (7상태 → 2) | 26,632 | 약 7,600 | 약 19,000 |
| BASEMUL TOTAL (9상태 → 2, 8회) | 9,232 | 약 2,200 | 약 7,000 |
| **합계** | | | **약 36,700** |

전체 105,286 사이클의 **약 35% 감소 → 약 68,600 사이클 / 686 µs**. 현재 PS `-O2` 소프트웨어(1,258 µs) 대비 1.13배에서 **약 1.8배**로 벌어집니다.

**비용과 위험.**
- 신규 DSP 불필요(기존 곱셈기를 채워 쓰는 것). 
- 추가 자원: 파이프라인 valid/인덱스 시프트 레지스터(스테이지당 약 20 FF × 스테이지 수, 총 100~200 FF 수준)와 드레인 제어 상태.
- **위험: 동시 활성 DSP 수가 늘어 라우팅 혼잡과 DSP 입력 경로의 팬인이 변하므로, WNS +0.105 ns인 현 설계에서 타이밍이 악화될 실질적 가능성이 있습니다. 재합성 없이 확인 불가 — 반드시 단계적으로 적용하고 매 단계 타이밍을 확인해야 합니다.** 권장 순서: BaseMul(의존성 없음, 뱅크 분리, 가장 안전) → NTT → INTT.
- 검증 부담: `tb_mlkem_poly_accelerator.sv`가 3개 커맨드를 모두 커버하므로 회귀 기반은 있으나, 파이프라인 드레인과 레이어 경계를 겨냥한 테스트를 추가해야 합니다.

**Priority: Medium** (대회 성과 관점에서는 가장 효과가 큰 항목이지만, 검증 완료된 설계를 건드리는 리스크가 있어 High로 올리지 않았습니다.)

---

### C-5. Watchdog의 `state <= REPORT`가 뒤따르는 `case`에 의해 덮어써짐

**파일:** `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:250-254, 278-341`

**문제.** 같은 `always_ff` 안에서 watchdog이 먼저 `state`를 쓰고, 이어서 `case(state)`가 다시 씁니다.

```systemverilog
            if (stage_timer >= MAX_STAGE_CYCLES-1) begin
                fault_detected_o <= 1;
                fault_code_o <= fault_code_o | F_TIMEOUT;
                state <= REPORT;
            end
            ...
            case (state)
                IDLE: if (launch) begin ...
                D_START: state <= D_WAIT;
```

SystemVerilog 절차적 대입은 마지막 대입이 승리하므로, `state`를 무조건 대입하는 상태(`D_START`, `T_GUARD`, `K_GUARD`, `C_GUARD`, `REPORT`, `default` — 13개 중 6개)에서는 **watchdog의 `REPORT` 전이가 무효화됩니다.**

현재는 기능적으로 문제가 없습니다. `fault_detected_o`가 별도로 래치되고 `T_START`/`K_START`/`C_START`가 `if (fault_detected_o || ...)`로 다시 검사하므로(`:298, 313, 328`) 결국 `REPORT`에 도달합니다. 그러나 이는 **우연한 안전성**이며, 향후 `*_GUARD`/`*_START` 구조를 바꾸면 조용히 깨집니다. 대회 심사나 기능 안전 검토에서 지적될 수 있는 구조입니다.

부수적으로 `stage_timer`는 상태 전이마다 0으로 리셋되므로(`:245-246`) **상태별 watchdog이며 전체 연산 watchdog이 아닙니다.** 13개 상태 × 300,000 = 최대 3.9 M 사이클(39 ms)까지 걸릴 수 있고, 매 사이클 상태가 바뀌는 병리적 진동은 영구히 검출되지 않습니다.

**개선 제안.**
1. watchdog을 명시적 최우선 순위로 재구성한다. `logic timeout_trip; always_comb timeout_trip = (stage_timer >= MAX_STAGE_CYCLES-1);` 를 두고 `if (timeout_trip) begin ... state <= REPORT; end else begin case(state) ... endcase end`. 로직 증가 없음.
2. 전체 연산 watchdog(`total_timer`)을 추가한다. 20비트로 최대 1,048,575 사이클이므로 105,286 사이클 연산에 약 10배 여유. 자원: FF 20개 + 20비트 비교기(약 8 LUT). 타이밍 영향 미미하나 FSM 다음 상태 로직에 팬인이 하나 늘므로 **재합성 확인 필요**.
3. `MAX_STAGE_CYCLES = 300000`의 근거(최장 상태 `D_WAIT` 약 98,400 사이클의 약 3배)를 파라미터 주석으로 남긴다.

**Priority: Medium**

---

### C-6. `mlkem_decaps_memory`의 `poly_mem` 주소가 배열 범위를 넘음

**파일:** `outputs/rtl_aead/rtl/mlkem_decaps_memory.sv:25, 45-53`

**문제.**
```systemverilog
    (* ram_style="block" *) logic [15:0] poly_mem[0:3071];
```
```systemverilog
        if(host_valid_i&&host_we_i&&host_region_i==2)
            poly_mem[host_addr_i]<=host_wdata_i[15:0];
        host_poly_q<=poly_mem[host_addr_i];
```

`host_addr_i`는 12비트(0~4095)인데 배열은 0~3071입니다. 프론트엔드의 `mem_addr`는 `mem_addr+12'd1`로 자동 증가하며(`mlkem_decaps_axi_lite_frontend.sv:89`) **상한 검사가 없습니다.** 따라서 호스트가 region 2에 3,072개를 넘게 쓰면 범위 밖 접근이 발생합니다. `poly_addr_i`(엔진 포트)도 동일하게 12비트입니다.

합성에서는 Vivado가 4,096 깊이 BRAM으로 올림 추론하여 실해가 없을 가능성이 높지만, **XSim에서는 범위 밖 쓰기가 무시되고 읽기는 `X`를 반환하여 X 전파를 일으킬 수 있습니다.** 즉 sim/synth 동작이 갈릴 수 있는 지점이며, H-1과 같은 계열의 위험입니다.

**개선 제안.**
1. 배열을 `poly_mem[0:4095]`로 선언해 주소 공간과 일치시킨다. BRAM 추론 결과가 바뀌지 않으므로(이미 4,096로 올림) **자원 증가 없음**.
2. 또는 `poly_mem[host_addr_i[11:0] % 3072]` 대신 명시적 상한 검사를 추가하고 초과 시 `RESP_SLVERR`를 반환한다(C-7과 함께 처리).
3. `sk_mem[host_addr_i[8:0]]`(512 중 408 사용), `ct_mem[host_addr_i[7:0]]`(256 중 192 사용)도 상한 검사 없이 wrap한다. 기능 침해는 아니나 잘못된 로드 시퀀스가 조용히 데이터를 겹쳐 쓰므로 검사 추가를 권장한다.

**Priority: Medium**

---

### C-7. 두 AXI-Lite 프론트엔드가 미매핑 주소에 SLVERR를 반환하지 않음

**파일:**
- `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:97, 99, 131-144, 249-256`
- `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv:50, 93, 109`

**문제.** 트래픽 프론트엔드는 응답이 항상 OKAY 고정입니다.

```systemverilog
    assign s_axi_bresp   = 2'b00;
    ...
    assign s_axi_rresp   = 2'b00;
```

쓰기 주소 디코드의 `default`(`:249-256`)는 `REG_INPUT_DATA`/`REG_INPUT_TAG` 범위 밖 주소를 조용히 버리고, 읽기 `default`(`:131-144`)는 0을 반환합니다. ML-KEM 프론트엔드도 `s_axi_rresp=RESP_OKAY` 고정(`:50`)이며 쓰기 `default`는 빈 블록(`:93`)입니다. 주소 정렬 검사도 없어 `0x005` 같은 비정렬 주소가 그냥 ACK됩니다.

ML-KEM 프론트엔드는 부분 strobe에 대해서만 SLVERR를 반환하므로(`:90`) 프로토콜 대응이 일관되지 않습니다. 드라이버 버그(오프셋 오타)나 악의적 PS 접근이 아무 신호 없이 무시되어 디버깅이 어렵습니다. 실제로 §B-5에서 발견한 `REG_CFG_SESSION_ID 0x028` 충돌이 바로 이 부류의 사고입니다.

**개선 제안.** 두 프론트엔드에 `decode_hit` 신호를 만들어 미스 시 `bresp`/`rresp`에 `2'b10`(SLVERR)을 반환한다. 자원: 주소 비교 결과 OR 리덕션(약 10 LUT 이하). `bresp`/`rresp`는 이미 레지스터로 만들 수 있으므로 **조합 깊이 증가 없음**.

**Priority: Medium**

---

### C-8. 트래픽 프론트엔드 읽기 경로가 비등록 주소 디코드 + 40:1 먹스

**파일:** `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:110-146, 199-201`

**문제.** 읽기 데이터 먹스가 `s_axi_araddr`를 직접 디코드하고, 그 결과가 같은 사이클에 `s_axi_rdata` 레지스터로 들어갑니다.

```systemverilog
            default: begin
                for (int rd = 0; rd < 16; rd++) begin
                    if (s_axi_araddr[8:0] == REG_INPUT_DATA + 4*rd)
                        read_mux = input_data[rd];
                    if (s_axi_araddr[8:0] == REG_OUTPUT_DATA + 4*rd)
                        read_mux = rsp_data_q[32*rd +: 32];
                end
                for (int rt = 0; rt < 4; rt++) begin
                    ...
```
```systemverilog
            if (s_axi_arvalid && s_axi_arready) begin
                s_axi_rdata  <= read_mux;
```

경로는 `인터커넥트 출력 레지스터 → 9비트 비교기 약 40개 → 32비트 40:1 우선순위 먹스 → s_axi_rdata`입니다. `rsp_data_q`(512비트)와 `input_data[16]`(512비트)를 모두 먹스 입력으로 받으므로 팬인이 큽니다. 또한 `for` 루프 안의 조건부 대입이 우선순위 체인을 만들어 병렬 먹스보다 깊어집니다.

**이 경로가 WNS 크리티컬 경로인지는 재합성 없이 확인 불가**이며, 개인적 추정으로는 Keccak 라운드 경로가 더 유력합니다. 다만 이 경로는 **개선하면 타이밍이 좋아지는 방향**이므로 여유 확보 후보로 기록합니다.

**개선 제안.**
1. `s_axi_araddr`를 먼저 레지스터에 받고 다음 사이클에 디코드/먹스한다. AXI-Lite는 읽기 지연에 제약이 없으므로 레이턴시 1사이클 증가는 허용된다. 조합 경로가 두 단으로 쪼개져 각 단이 짧아진다.
2. `for` 루프의 조건부 대입을 `case` 또는 인덱스 계산 기반 병렬 먹스로 바꾼다: `if (araddr inside [REG_OUTPUT_DATA : REG_OUTPUT_DATA+63]) read_mux = rsp_data_q[32*araddr[5:2] +: 32];` 형태로 40개 비교기를 범위 비교 4개 + 4비트 셀렉트로 대체. **LUT 감소 + 깊이 감소** 양쪽에 유리.
3. 레이턴시 변화가 `tb_mlkem_secure_channel_complete_axi_top.sv`의 `ar` 태스크 타이밍 가정을 깨는지 확인이 필요하다.

**Priority: Medium**

---

## D. 우선순위 Medium — 테스트벤치 / 검증

### D-1. 합성 closure 안의 모듈 중 전용 TB가 없는 것들

**파일:** `outputs/rtl_aead/tb/` 전체

**문제.** 인스턴스 그래프로 계산한 합성 closure(`mlkem_secure_channel_complete_axi_top` 기준, §F-2)와 TB의 DUT 목록을 교차 확인한 결과, 다음 모듈에 **전용 TB가 없습니다.**

| 모듈 | 파일 | 현재 커버리지 |
|---|---|---|
| `mlkem_secure_channel_fault_protected_indexed_axi_top` | `..._indexed_axi_top.sv` | 통합 TB 경유만. fault 7종 미검증 (H-5) |
| `aead_traffic_indexed_axi_lite_frontend` | 동명 파일 | 통합 TB에서 슬롯 0만 (H-3) |
| `secure_channel_material_bram_core` | 동명 파일 | 통합 TB 경유만 |
| `mlkem512_kpke_reencrypt_shared_engine` | 동명 파일 | 없음 (비-shared 변형에만 TB 존재) |
| `keccak_f1600` | 동명 파일 | `tb_sha3_shake_stream` 간접만 |
| `aead_fixed64_engine` | 동명 파일 | 세션 매니저/통합 TB 경유 (TB는 wrapper 변형 대상) |
| `mlkem_poly_tomsg_controller` | `mlkem_poly_addsub_controller.sv:93` | **없음** (H-2) |
| `mlkem_encode12_group` | `mlkem512_codec_primitives.sv:73` | 없음 (같은 파일의 다른 6개는 커버됨) |

전반적 패턴이 뚜렷합니다: **TB 스위트가 비-shared / 비-indexed / wrapper 변형을 겨냥하는 동안 합성 closure는 shared / indexed / engine 변형을 씁니다.** `tb_aead_fixed64.sv`는 `aead_fixed64_wrapper`와 `aead_fixed64_decrypt_wrapper`를 DUT로 쓰는데 둘 다 closure 밖이고, `tb_secure_channel_core.sv`의 `secure_channel_core`, `tb_mlkem512_decaps_engine.sv`의 `mlkem512_decaps_engine`, `tb_mlkem512_kpke_reencrypt_engine.sv`의 `mlkem512_kpke_reencrypt_engine`도 모두 closure 밖입니다.

**개선 제안.**
1. `keccak_f1600` 전용 TB를 FIPS 202 KAT(빈 문자열, 그리고 전-0/전-1 1600비트 상태 입력)으로 신설한다. 크리티컬 경로 최유력 후보이자 모든 해시의 기반이므로 직접 KAT이 필수적이다.
2. `aead_traffic_indexed_axi_lite_frontend` 단위 TB를 신설하고 다음을 커버한다: 슬롯 0/1/62/63 왕복, 슬롯 64/127 쓰기 시 앨리어싱 문서화(H-3), `pending`/`inflight` 중 START 재발행 무시, 미매핑 주소 접근(C-7), `wstrb` 부분 쓰기, `inflight` 중 `input_data` 덮어쓰기.
3. 각 TB 파일 상단에 "이 TB가 검증하는 모듈은 합성 closure에 포함됨/미포함됨"을 한 줄로 명시해 혼동을 막는다.
4. closure 밖 13개 변형 파일의 유지 여부를 결정한다(§F-2). 유지한다면 어떤 구성에서 쓰이는지 `rtl/README.md`에 명시하고, 아니면 `rtl/legacy/`로 분리해 TB와 짝을 맞춘다.

**Priority: Medium**

---

### D-2. TB 4개가 시뮬레이터 오류 카운트를 올리지 않음

**파일:**
- `outputs/rtl_aead/tb/tb_aead_fixed64.sv:141-242`
- `outputs/rtl_aead/tb/tb_aead_arbiter_4session.sv`, `tb_aead_arbiter_rr4_extended.sv`, `tb_aead_axi_lite_wrapper.sv`

**문제.** 31개 TB 중 4개가 `$error`/`$fatal`을 전혀 사용하지 않습니다. 특히 `tb_aead_fixed64.sv`는 **ChaCha20과 Poly1305의 유일한 KAT 테스트**인데 실패를 `$display`로만 알립니다.

```systemverilog
        if (cycles >= 100 || chacha_result !== expected_keystream) begin
            $display("FAIL: ChaCha20 counter=1 block mismatch");
```
```systemverilog
        if (failures == 0)
            $display("ALL RTL TESTS PASSED");
        else
            $display("RTL TESTS FAILED: %0d failure(s)", failures);
```

`$finish` 전 `$fatal`이 없으므로 **XSim 종료 코드가 0이고 오류 카운트도 0**입니다. 로그를 문자열로 grep하지 않는 회귀 스크립트는 ChaCha20/Poly1305 KAT 실패를 통과로 집계합니다. 나머지 3개(아비터 2개, wrapper 1개)는 애초에 자동 판정 코드가 없어 파형 육안 검사용으로 보입니다.

또한 `cycles >= 100`/`>= 300`/`>= 500` 같은 타임아웃과 값 불일치가 **같은 조건에 OR로 묶여** 있어, 실패 시 "타임아웃"인지 "값 오류"인지 로그만으로 구분되지 않습니다.

**개선 제안.**
1. `tb_aead_fixed64.sv`의 마지막을 `if (failures != 0) $fatal(1, "RTL TESTS FAILED: %0d", failures);`로 바꾼다. 개별 FAIL 지점도 `$error`로 바꿔 시뮬레이터 오류 카운트에 반영시킨다.
2. 타임아웃과 값 불일치를 분리해 서로 다른 메시지를 내도록 한다.
3. 아비터/wrapper TB 3개는 (a) 자동 판정을 추가하거나 (b) `rtl/README.md`에 "파형 검사 전용, 회귀 대상 아님"으로 명시한다. closure 밖 모듈 대상이므로 (b)도 합리적 선택이다.

**Priority: Medium**

---

### D-3. 64세션 검증의 키 분리 판정이 키스트림 재사용을 잡지 못함

**파일:** `outputs/rtl_aead/ps_driver/pqc_64session_validation.c:88-92, 106-107`

**문제.**
```c
        if (slot != 0u
            && !bytes_differ(ciphertext, previous_ciphertext,
                             AEAD_HW_PACKET_BYTES)
            && !bytes_differ(tag, previous_tag, AEAD_HW_TAG_BYTES))
            distinct_ok = 0;
```

`distinct_ok`가 0이 되는 것은 **암호문과 태그가 둘 다 같을 때뿐**입니다. 따라서 "키스트림은 동일하지만 태그만 다른" 상태 — 즉 **정확히 §B-4가 지적한 논스/키 재사용 실패 모드** — 에서 이 테스트는 **PASS로 보고**합니다. 태그는 AAD에 session_id가 들어가므로 키가 같아도 달라지기 때문입니다.

현재 설계는 session_id가 transcript를 통해 키 파생에 반영되므로(§B-4에서 확인) 실제로는 암호문도 달라집니다. 즉 지금은 오탐/미탐이 없습니다. 그러나 **판정 기준이 검증하려는 속성보다 약해서, 회귀가 발생해도 감지하지 못합니다.** README가 이 테스트 결과를 "세션 ID별 트래픽 키 분리" 근거로 인용하고 있으므로 보강 가치가 있습니다.

**개선 제안.**
```c
        if (slot != 0u
            && !bytes_differ(ciphertext, previous_ciphertext,
                             AEAD_HW_PACKET_BYTES))
            distinct_ok = 0;
```
키가 다르면 같은 평문·같은 카운터에서도 암호문이 반드시 달라야 하므로, 암호문 단독 비교가 올바른 판정이다. 추가로 슬롯 전체의 첫 암호문을 배열에 보관해 **전수 쌍 비교**(64×63/2 = 2,016회 memcmp, 부담 없음)를 수행하면 인접 슬롯만 비교하는 현재 방식의 구멍도 막힌다.

**Priority: Medium**

---

### D-4. 드라이버가 오류 반환 시 호출자 출력 버퍼를 그대로 남김

**파일:** `outputs/rtl_aead/ps_driver/aead_hw.c:179-184, 220-225`

**문제.**
```c
    if (wait_status(device, STATUS_DONE, 0u, &status) != AEAD_HW_OK)
        return AEAD_HW_ERR_TIMEOUT;
    if ((status & STATUS_AUTH_OK) == 0u || (status & STATUS_ERROR) != 0u)
        return AEAD_HW_ERR_AUTH;
    read_packet(device, plaintext);
```

인증 실패/타임아웃 시 `plaintext`를 건드리지 않고 반환합니다. 하드웨어는 `REG_OUTPUT_DATA`를 0으로 만들지만 드라이버가 읽지 않으므로, **호출자 버퍼에는 직전 호출의 평문이 남습니다.** 반환값을 무시하는 호출자는 오래된 평문을 새 수신 데이터로 처리합니다. `aead_hw_encrypt`의 `ciphertext`/`tag`도 동일합니다.

현 상위 호출자는 방어하고 있습니다 — `uart_secure_demo.c:288`이 `memset(plaintext, 0, sizeof(plaintext))`를 먼저 수행합니다. 즉 **지금은 사고가 없지만, 방어가 호출자에 있고 API 계약에는 없습니다.**

**개선 제안.** 모든 오류 반환 경로 앞에서 출력 버퍼를 0으로 채운다.

```c
    if (wait_status(device, STATUS_DONE, 0u, &status) != AEAD_HW_OK) {
        memset(plaintext, 0, AEAD_HW_PACKET_BYTES);
        return AEAD_HW_ERR_TIMEOUT;
    }
```
`aead_hw.h`에 "오류 반환 시 출력 버퍼는 0으로 채워진다"를 계약으로 명시한다.

**Priority: Medium**

---

### D-5. PS MMIO 접근에 메모리 배리어가 없음

**파일:**
- `outputs/rtl_aead/ps_driver/aead_hw.c:37-50`
- `outputs/rtl_aead/ps_driver/mlkem_decaps_hw.c` (`wr`/`rd`)

**문제.**
```c
static void mmio_write(const aead_hw_t *device, uint32_t offset,
                       uint32_t value)
{
    volatile uint32_t *address =
        (volatile uint32_t *)(device->base_address + (uintptr_t)offset);
    *address = value;
}
```

Xilinx 표준 `Xil_Out32`/`Xil_In32`가 아니라 생 `volatile` 포인터를 씁니다. `volatile`은 **컴파일러 재정렬만** 막고 프로세서/버스 레벨 재정렬은 막지 못합니다. 문제가 되는 시퀀스:

```c
    write_packet(device, message, message_length);   /* 16 워드 쓰기 */
    mmio_write(device, REG_CONTROL, CONTROL_START);  /* 시작 */
```
```c
    r=load(d,REGION_CT,ct,MLKEM512_CIPHERTEXT_BYTES);if(r)return r;
    wr(d,REG_SLOT,slot);wr(d,REG_SESSION,session_id);wr(d,REG_CONTROL,1u<<8);
    wr(d,REG_CONTROL,1u);
```

데이터 쓰기가 START 쓰기 **앞에** 완료되어야 합니다. Vitis 표준 BSP가 PL AXI GP 영역을 Device(strongly-ordered)로 매핑하므로 실제로는 재정렬되지 않아 동작합니다. 그러나 **BSP의 MMU 속성 테이블에 암묵적으로 의존**하는 구조이며, 매핑을 Normal-Noncacheable로 바꾸거나 캐시 설정을 조정하면 조용히 깨집니다. 이런 버그는 최적화 레벨에 따라 간헐적으로 나타나 추적이 매우 어렵습니다.

**개선 제안.** `mmio_write`/`mmio_read`/`wr`/`rd`를 `Xil_Out32`/`Xil_In32`로 교체한다(내부에서 `dmb`를 발행). 직접 구현을 유지하려면 START 쓰기 직전에 `__asm__ volatile("dmb" ::: "memory");`를 명시적으로 넣는다. 런타임 비용은 트랜잭션당 수 사이클이며, AEAD 64바이트 작업의 21 µs 예산에서 무의미한 수준이다.

**Priority: Medium**

---

## E. 검증 결과 — 문제가 없음을 확인한 항목

리뷰 요청 항목 중 **의심했으나 코드 확인 결과 올바른** 것들입니다. 재작업 대상이 아님을 분명히 하기 위해 기록합니다.

### E-1. 태그 비교는 상수 시간이며 조기 종료가 없음

`aead_fixed64_engine.sv:52`:
```systemverilog
    assign tag_matches = ~(|(poly_tag ^ received_tag_reg));
```
128비트 전체 XOR 후 OR 리덕션으로, 바이트 단위 조기 종료가 없습니다. **타이밍 누출 없음.**

인증 실패 경로(`:155-161`)가 성공 경로(`:151-154` → `S_WAIT_DECRYPT_STREAM`, ChaCha20 41사이클)보다 빨리 끝나므로 관측 가능한 시간 차이는 존재합니다. 그러나 이 차이가 드러내는 정보는 `auth_ok_o` 출력이 이미 공개하는 것과 동일하므로 **추가 누출이 아닙니다.**

### E-2. Poly1305의 최종 축약과 리밋 폭이 정확함

`poly1305_fixed96.sv:247-266`의 단일 조건부 감산이 충분한지 경계 분석했습니다. `S_NORM4`에서 `d_acc[4] &= LIMB_MASK`이므로 `h_limb[4] < 2^26`, 따라서 `packed_limb4 = h_limb[4] + carry <= 2^26`입니다. `h_value_reg` 최댓값은 `2^130 + 2^104 - 1`이고, `P130 = 2^130 - 5`를 한 번 감산하면 `2^104 + 4 < P130`이 되어 **정규 형태가 보장됩니다.** `d_acc`(64비트)는 최대 항 `28×29 = 57`비트를 5개 누산해 60비트이므로 여유가 있고, `r5_limb`(29비트)는 `(2^26-1)×5 < 2^29`에 맞습니다. **모두 정확합니다.**

### E-3. Montgomery / Barrett 상수가 참조 구현과 일치

`mlkem_poly_accelerator.sv:220-225`의 Montgomery는 `t = (int16_t)(a*QINV); r = (a - t*q) >> 16` 구조로 `QINV = 62209`, `q = 3329`가 참조 구현과 일치합니다. `:270-282`의 Barrett은 `v = 20159`, 라운딩 상수 `32'sd33554432 = 2^25`, 시프트 26비트로 `v = ((1<<26) + q/2)/q` 정의와 일치합니다. **정확합니다.**

### E-4. Poly1305 길이 블록 하드코딩은 버그가 아님

`aead_fixed64_engine.sv:116, 132`:
```systemverilog
                            poly_message_reg[767:640] <= {64'd64, 64'd16};
```
AAD 길이 16, 암호문 길이 64가 상수입니다. 처음에는 `req_data_len_i`를 무시하는 버그로 의심했으나, PC 클라이언트가 항상 64바이트로 제로 패딩합니다.

`uart_secure_client.py:170-174`:
```python
    padded = plaintext + bytes(PACKET_BYTES - len(plaintext))
    request = ChaCha20Poly1305(state["tx_key"]).encrypt(
        build_nonce(state["tx_prefix"], tx_counter),
        padded,
        build_aad(state["sid"], tx_counter, len(plaintext)),
    )
```

즉 AEAD는 항상 정확히 64바이트에 대해 수행되고, 실제 길이는 **AAD를 통해 인증**됩니다(`aead_session_manager_bram.sv:376`의 `engine_aad[103:96] <= active_len_q`). 양측 규약이 일치하며 길이 혼동 공격도 AAD로 차단됩니다. **고정 64바이트 패킷 설계에 부합하는 올바른 구현입니다.** (RFC 8439 상호운용성 관점에서는 이탈이므로, 외부 라이브러리와 직접 연동할 계획이 있으면 문서화가 필요합니다.)

### E-5. 세션 ID가 키 파생에 반영됨 — 슬롯 간 키 분리 성립

같은 KAT 암호문으로 64슬롯을 설치하는 검증 플로우에서 키가 동일해지는지 추적했습니다. `mlkem_shared_hash_engine.sv:94-95`의 `C_TRANSCRIPT` 입력 마지막 4바이트가 `sid`이고, `C_KDF`는 `("ZYNQ-PQCC-v1" || ss || transcript)`를 입력받으므로 **session_id가 transcript를 경유해 트래픽 키에 확실히 반영됩니다.** PC 측(`uart_secure_client.py:34-36`)도 동일하게 계산합니다. 슬롯 간 키 분리는 **성립합니다.** (다만 키 유일성이 PS의 session_id 관리에만 의존하는 문제는 §B-4에 별건으로 기록.)

### E-6. 재설치 시 카운터 초기화가 RTL에서 실제로 동작

`aead_session_manager_bram.sv:138-139`의 `else config_word = 32'd0`가 `WORD_TX_COUNT_LO`(19) ~ `WORD_RESERVED`(23)를 0으로 덮고, `S_CFG_WRITE`가 워드 0~23 전체를 순회하므로(`:311-319`) **README의 "슬롯 재사용 시 counter 초기화" 주장은 RTL에서 확인됩니다.** (전용 소거 경로가 없다는 별건 문제는 §B-1.)

### E-7. CDC 동기화기 누락 없음 — 단일 클록 도메인

설계 전체가 `aclk`(PS FCLK_CLK0) 단일 도메인입니다. `system_processing_system7_0_0.xci:771`에서 `PCW_FPGA0_PERIPHERAL_FREQMHZ = 100`을 확인했습니다(블록 디자인의 `rst_ps7_0_50M` 인스턴스명은 100 MHz 설계에 남은 옛 이름일 뿐 실제 주파수와 무관 — §F-6).

리셋 해제 동기화기가 올바르게 구현되어 있습니다. `mlkem_secure_channel_complete_axi_top.sv:28-35`:
```systemverilog
  (* ASYNC_REG="TRUE" *) logic[1:0]rst_sync_q;
  always_ff @(posedge aclk or negedge aresetn)
    if(!aresetn) rst_sync_q<=2'b00;
    else         rst_sync_q<={rst_sync_q[0],1'b1};
  assign rst_sync_n=rst_sync_q[1];
```
2단 플롭 + `ASYNC_REG` 속성으로 비동기 리셋 해제를 동기화합니다. 데이터 CDC 경로가 없으므로 **추가 동기화기가 필요한 지점은 발견되지 않았습니다.** `fault_inject_i`는 유일한 비동기 후보였으나 블록 디자인에서 상수 0으로 묶여 있습니다(§F-5).

### E-8. NTT/INTT의 dual-port 동시 쓰기가 안전

`mlkem_poly_accelerator.sv:156-160`의 `NTT_WRITE`가 `a_addr0 = butterfly`, `a_addr1 = butterfly+span`에 동시 쓰기를 수행합니다. `span`은 NTT에서 128→2, INTT에서 2→128 범위이므로 **항상 `span >= 2`이고 두 주소가 절대 같아지지 않습니다.** 동일 주소 동시 쓰기(정의되지 않은 동작)는 발생하지 않습니다. 단 이 안전성이 레이어 수(7)에 암묵적으로 의존하므로 `assert (span >= 2)` 또는 주석 추가를 권장합니다(§G-7).

### E-9. Fail-closed 출력 de-assert가 실제로 동작

§B-3 표에 정리한 대로, fault 발생 시 `rsp_auth_ok_o` de-assert, `rsp_data_o` 0 마스킹, `rsp_error_o` assert, `install_req` 차단, `final_fail` 보고가 모두 RTL에서 확인됩니다. **"제어 출력이 실제로 de-assert 되는가"에 대한 답은 예입니다.** 잔여 문제는 세션 카운터 오염(§B-3)과 PS 가시성 부재(§B-5)입니다.

---

## F. 우선순위 Low — IP 패키징 / 저장소 위생

### F-1. `outputs/rtl_aead/rtl`과 패키지 IP는 실질 동기 상태 — 단 무보호

**파일:**
- `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:430`
- `zed_pqc/ip_repo/secure_channel_ip/src/aead_session_manager_bram.sv:431`
- 동일 패턴: `secure_channel_material_bram_core.sv`

**문제.** README가 "문서화된 footgun"으로 지목한 이중 사본 드리프트를 전수 diff한 결과, **A(`outputs/rtl_aead/rtl`)와 B(패키지 IP) 사이의 차이는 두 파일의 말미 빈 줄 1개뿐**입니다.

```
--- outputs/rtl_aead/rtl/aead_session_manager_bram.sv
+++ zed_pqc/ip_repo/secure_channel_ip/src/aead_session_manager_bram.sv
@@ -428,3 +428,4 @@
         end
     end
 endmodule
+
```

기능 차이 없음. C(`project_1.ipdefs`)는 B와 바이트 단위로 동일하며 `component.xml`도 동일합니다. **즉 이 축의 동기화는 잘 유지되고 있습니다.** 실제 드리프트는 D(시뮬레이션 사본)에서 발생했고 H-1에 기록했습니다.

문제는 이 동기화가 **전적으로 수작업**이라는 점입니다. 4개 사본 어디에도 체크섬/CI 검사가 없고, `component.xml`의 `viewChecksum`(`:396`의 `d2671c9a`)은 Vivado 내부 값이라 사용자 수정을 잡아주지 않습니다.

**개선 제안.** 4개 사본의 SHA-256 일치를 검사하는 스크립트를 `scripts/`에 추가하고 커밋 훅 또는 CI에서 실행한다. `component.xml`의 파일 목록을 파싱해 대상을 자동 산출하면 파일 추가 시에도 유지된다. 공백 차이는 `--ignore-trailing-space`로 허용하되 기능 차이는 실패시킨다.

**Priority: Low** (동기화 자체는 정상이나 재발 방지 장치가 없음)

---

### F-2. 합성 closure 밖 파일이 IP fileset과 RTL 디렉터리에 혼재

**파일:** `zed_pqc/ip_repo/secure_channel_ip/component.xml` (fileset 27개), `outputs/rtl_aead/rtl/` (40개 파일 / 56개 모듈)

**문제.** `component.xml:388`의 `modelName = mlkem_secure_channel_complete_axi_top`을 기준으로 인스턴스 그래프를 전개한 결과, 합성 closure는 **23개 파일**입니다. 따라서:

**IP fileset에 있으나 closure 밖(4개, 합성 시 pruning되어 자원은 소비하지 않음):**
`aead_arbiter_4session.sv`, `aead_traffic_axi_lite_frontend.sv`, `mlkem_secure_channel_fault_protected_axi_top.sv`, `secure_channel_material_core.sv`
— 뒤 두 개는 `..._fault_protected_axi_top → secure_channel_material_core → aead_arbiter_4session` 서브트리를 이루는 4세션 변형입니다. H-5의 공격 TB가 겨냥하는 바로 그 계층입니다.

**`outputs/rtl_aead/rtl`에 있으나 IP fileset에도 closure에도 없는 것(13개):**
`aead_axi_lite_wrapper.sv`, `aead_fixed64_wrapper.sv`, `aead_fixed64_decrypt_wrapper.sv`, `secure_channel_core.sv`, `mlkem_session_kdf.sv`, `mlkem_hash_g.sv`, `mlkem_matrix_poly_generator.sv`, `mlkem_noise_poly_generator.sv`, `mlkem_handshake_transcript_hash.sv`, `mlkem512_decaps_engine.sv`, `mlkem512_kpke_reencrypt_engine.sv`, `mlkem_secure_channel_axi_top.sv`, `mlkem_secure_channel_shared_axi_top.sv`

**중요: 누락된 파일은 없습니다.** closure 23개 파일이 모두 IP fileset에 존재하므로 **합성 실패 위험은 없습니다.** 문제는 어느 파일이 실제 비트스트림에 들어가는지 저장소에서 판별할 수 없다는 점이며, 이것이 H-5(TB가 잘못된 top을 겨냥)와 D-1(TB가 변형 모듈을 겨냥)의 근본 원인입니다.

**개선 제안.**
1. `outputs/rtl_aead/rtl/README.md`에 파일별로 "출하 closure / 대체 구성 / 레거시"를 표로 명시한다.
2. closure 밖 13개 파일을 `outputs/rtl_aead/rtl/variants/`로 이동하거나, 최소한 각 파일 헤더에 `/* NOT IN SHIPPED BITSTREAM — see rtl/README.md */`를 추가한다.
3. IP fileset의 4개 비-closure 파일은 제거를 검토한다. 유지 시 재패키징마다 불필요한 elaboration 시간이 들고, 실수로 top을 바꿨을 때 조용히 다른 설계가 합성될 여지가 남는다.
4. §F-1의 체크섬 스크립트에 "closure 파일 목록 산출" 기능을 함께 넣으면 문서와 실제가 어긋나는 것을 자동 탐지할 수 있다.

**Priority: Low**

---

### F-3. `validate_ip.tcl`이 타 개발자 머신의 절대 경로를 하드코딩

**파일:** `zed_pqc/ip_repo/secure_channel_ip/validate_ip.tcl:1`

```tcl
set core_dir {C:/Users/bomin/Documents/Codex/2026-08-10/new-chat/zed_pqc/ip_repo/secure_channel_ip}
```

다른 환경에서 그대로 실행하면 실패합니다. IP 무결성 검사가 재현 절차의 일부라면 경로를 스크립트 위치 기준 상대 경로로 바꿔야 합니다.

**개선 제안.**
```tcl
set core_dir [file dirname [file normalize [info script]]]
```

**Priority: Low**

---

### F-4. XDC 제약 파일이 저장소에 없음

**파일:** `constraints/` (`.gitkeep`만 존재), 저장소 전체 `*.xdc` 0건

**문제.** `find . -name "*.xdc"` 결과가 0건입니다. `constraints/`는 `.gitkeep`만 있는 빈 디렉터리입니다. README가 내세우는 **WNS +0.105 ns / TNS 0.000 ns / WHS +0.007 ns는 저장소만으로 재현할 수 없습니다.** 클록 정의가 블록 디자인의 PS7 IP에서 자동 생성되는 구조라면 별도 XDC가 불필요할 수 있으나, `max_fanout`/`ASYNC_REG` 같은 속성이 RTL 인라인으로만 존재하는 점(`sha3_shake_stream.sv:32, 70`)을 보면 물리 제약이 전혀 없는 상태로 타이밍을 맞춘 것으로 보입니다.

**개선 제안.** 최종 구현 런에서 사용된 XDC(자동 생성분 포함)를 `constraints/`에 내보내 커밋하고, 타이밍 리포트(`*.rpt`는 `.gitignore` 대상이므로 요약만)를 `docs/`에 남긴다. WNS 여유가 0.105 ns뿐이므로 **어떤 RTL 변경이든 재합성 전후 비교가 필수**이며, 기준 제약이 저장소에 없으면 비교 자체가 불가능하다.

**Priority: Low** (단, §A/§C의 모든 RTL 변경 제안을 실행하려면 **선행 조건**)

---

### F-5. `fault_inject_i` 테스트 포트가 출하 IP에 노출되고 `F_OUTPUT` 검출기가 사실상 dead

**파일:**
- `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:29, 199-204, 271-274, 288, 307, 322`
- `zed_pqc/project_64session/project_1.srcs/sources_1/bd/system/ip/system_xlconstant_0_0/system_xlconstant_0_0.xci`

**문제.** 블록 디자인에서 `fault_inject_i`는 6비트 상수 0으로 묶여 있습니다(`CONST_VAL=0`, `CONST_WIDTH=6`, `system.bd:2331-2334`의 `xlconstant_0_dout → mlkem_secure_channel_0/fault_inject_i`). 따라서 합성 시 6개 주입 탭이 전부 상수 폴딩됩니다.

```systemverilog
        hcmd_hash = hcmd_req ^ {2'b00, fault_inject_i[0]};   /* → hcmd_req */
        hdone_ctl = hdone_raw & ~fault_inject_i[4];          /* → hdone_raw */
        mon_rsp_valid = raw_rsp_valid | fault_inject_i[5];   /* → raw_rsp_valid */
```

**긍정적 확인:** 주입 탭은 자원과 타이밍을 전혀 소비하지 않습니다.

**부정적 귀결:** `mon_rsp_valid`가 `raw_rsp_valid`로 축약되므로 `F_OUTPUT` 검출기가 도달 불가능해집니다.

```systemverilog
            if (mon_rsp_valid && !outstanding) begin
                fault_detected_o <= 1;
                fault_code_o <= fault_code_o | F_OUTPUT;
            end
```

`raw_rsp_valid`는 요청이 outstanding일 때만 어서트되므로 이 조건은 출하 비트스트림에서 **영구히 거짓**입니다. 즉 7종 fault 중 1종이 하드웨어에서 죽어 있습니다.

또한 시뮬레이션에서 `fault_inject_i[5]`를 펄스하면 `rsp_valid_o`가 위조되어 프론트엔드가 `inflight`를 내리고, 이후 세션 매니저가 `S_RESPONSE`에서 `rsp_ready_i = inflight = 0`으로 영구 대기하는 **데드락**이 가능합니다. 출하 비트스트림에서는 상수 0이라 도달 불가하므로 시뮬레이션 전용 주의사항입니다.

**개선 제안.**
1. `F_OUTPUT`을 살리려면 `fault_inject_i`와 무관한 실제 감시 조건으로 바꾼다(예: 세션 매니저 `rsp_valid`가 `outstanding` 없이 뜨는 경우를 매니저 내부 신호로 직접 감시).
2. `fault_inject_i`를 합성 파라미터(`parameter bit ENABLE_FAULT_INJECT = 0`)로 감싸 프로덕션 빌드에서 포트 자체가 사라지게 하고, TB 빌드에서만 활성화한다. 포트를 남겨둘 경우 누군가 GPIO로 재배선하면 `[5]`가 DoS 벡터가 된다.
3. 시뮬레이션 데드락을 H-5의 이식된 TB에 네거티브 케이스로 명시 기록한다.

**Priority: Low**

---

### F-6. 리셋 스타일 혼재 및 기타 명명 위생

**혼재된 리셋 스타일.** `aead_traffic_indexed_axi_lite_frontend.sv:148`은 동기 리셋입니다.
```systemverilog
    always_ff @(posedge clk_i) begin
        if (!rst_ni) begin
```
같은 클록 도메인의 나머지 모듈은 모두 비동기 리셋입니다(예: `aead_session_manager_bram.sv:209`의 `always_ff @(posedge clk_i or negedge rst_ni)`). `mlkem_decaps_axi_lite_frontend.sv:44-46`은 이 도메인의 관례가 비동기임을 주석으로 못박고 있는데, 트래픽 프론트엔드만 벗어나 있습니다. `rst_sync_n`이 동기화된 신호이고 FCLK가 항상 동작하므로 기능 문제는 없으나 일관성 위반입니다. **Priority: Low**

**`rst_ps7_0_50M` 인스턴스명.** `system.bd:1871-1875`의 `proc_sys_reset` 인스턴스가 100 MHz 설계에서 `50M` 이름을 유지하고 있습니다. `slowest_sync_clk`는 FCLK_CLK0(100 MHz)에 올바르게 연결되어 있으므로 **기능 문제 없음, 명명만 혼동 유발.** **Priority: Low**

**`zetas` ROM 이중화.** `mlkem_poly_accelerator.sv:82-115`의 `initial` 블록 ROM이 `zetas[zeta_index]`(`:216, 269`)와 `zetas[64+(pair_index>>1)]`(`:310-311`)의 두 독립 인덱스로 읽히므로 Vivado가 ROM을 두 벌 추론할 가능성이 높습니다(각 128×16비트, 약 128 LUT). 인덱스를 먹스로 합치면 약 128 LUT 절감. 또한 `rom_style` 속성이 없고 `initial` 기반이라 도구 의존적입니다. `localparam logic signed [15:0] ZETAS [0:127] = '{...}`로 바꾸는 것이 더 견고합니다. **Priority: Low**

**`out_index` 포화가 오류를 숨김.** `mlkem_shared_hash_engine.sv:112`의 `if(out_index<7'd71)out_index<=out_index+7'd1;`은 출력이 72바이트를 넘으면 `digest_o[575:568]`을 조용히 덮어씁니다. `error_o`를 세우는 것이 맞습니다. **Priority: Low**

**`ct_addr_o` 언더플로우(무해).** `mlkem_shared_hash_engine.sv:52`의 `ct_addr_o=(in_index-32)>>2`는 `in_index<32`일 때 무부호 wrap하지만, 해당 구간은 FSM이 비메모리 경로를 타므로(`memory_input()`이 `n>=32`를 요구) 결과가 사용되지 않습니다. **기능 영향 없음**, 가독성 차원에서 조건부로 감싸는 것을 권장. **Priority: Low**

**RX 카운터 재동기 수단 없음.** `aead_session_manager_bram.sv:357`의 `request_counter_q != active_rx_counter`는 엄격한 순차 일치를 요구합니다(슬라이딩 윈도우 없음). 재전송 거부는 확실하지만 패킷 1개만 유실되어도 해당 슬롯 수신 방향이 영구 불능입니다. UART 링크는 신뢰성이 있어 현재 문제가 되지 않으므로 무선 확장 시 검토 항목으로만 기록합니다. **Priority: Low**

**슬롯 주소 산술.** `aead_session_manager_bram.sv:121-124`의 `slot_base = slot * WORDS_PER_SESSION`(24)은 6비트 × 상수 곱셈입니다(`slot<<4 + slot<<3`). `mlkem_shared_hash_engine.sv:64`의 `poly_addr_o=slot*256+coeff_count`와 `mlkem_poly_addsub_controller`의 `slot*256+index`는 `{slot, index}` 연접으로 대체 가능합니다(H-1에서 이미 한 사본에 적용됨). 모두 레지스터 출력에서 출발하는 경로이며 Vivado가 대개 최적화하므로 영향은 작습니다. **Priority: Low**

---

### F-7. PS 측 기타 위생

**`pqc_session_scheduler` 전체가 미사용.** `grep -rln "pqc_scheduler"` 결과 `pqc_session_scheduler.c/.h`와 `README.md`뿐입니다. `zed_pqc_bringup.c:126-154`의 `main`은 `pqc_64session_validation_run`과 `uart_secure_demo_run`만 호출합니다. `aead_session_manager_bram.sv:6-10`의 주석이 "fairness/queuing은 PS 소프트웨어 책임"이라고 명시했는데 **그 책임을 지는 코드가 빌드에 들어가지 않습니다.** 제거하거나 실제로 배선해야 합니다. **Priority: Low**

**`first_set_bit(0)`이 무한 루프.** `pqc_session_scheduler.c:10-18`:
```c
static uint8_t first_set_bit(uint32_t value)
{
    uint8_t bit = 0u;
    while ((value & 1u) == 0u) {
        value >>= 1;
        ++bit;
    }
    return bit;
}
```
`value == 0`이면 `value`가 0으로 고정되어 루프가 끝나지 않습니다. 세 호출처(`:74, 77, 83`)가 모두 0이 아님을 보장하므로 **현재 도달 불가**하나, 무방비 프리미티브입니다. `if (value == 0u) return 0u;`를 추가하고, 라운드로빈 스캔이 2워드(64세션) 및 2의 거듭제곱을 전제한다는 점(`:68`의 `start_word ^ 1u`, `:90`의 `& (PQC_SCHEDULER_SESSIONS - 1u)`)을 `static_assert` 또는 주석으로 고정한다. **Priority: Low**

**`main` 심볼 중복.** `zed_pqc_bringup.c:126`과 `example_baremetal.c:10`이 모두 `main`을 정의합니다. 디렉터리 전체를 컴파일하면 링크 오류가 납니다. Vitis 프로젝트에서 한쪽을 제외하고 있다면 그 사실을 `ps_driver/README.md`에 명시해야 합니다. **Priority: Low**

**`aead_hw_decrypt`가 응답 슬롯을 검증하지 않음.** `REG_RSP_SLOT`(0x028)이 응답의 슬롯을 반환하는데 드라이버는 읽지 않습니다. 단일 outstanding 구조이므로 현재 위험은 없으나, 응답/요청 대응을 확인하면 방어가 강해집니다. **Priority: Low**

---

## G. RTL 엔지니어 인계 항목

RTL 코드 변경이 필요한 항목만 추렸습니다. 각 항목은 `fpga-rtl-engineer`가 바로 착수할 수 있도록 대상 파일·라인, 변경 내용, 검증 조건, 타이밍/자원 리스크를 함께 적었습니다.

### G-0. 선행 조건 (RTL 수정 전 반드시 완료)

1. **§F-4** — 현재 구현 런의 XDC와 타이밍 요약을 `constraints/`, `docs/`에 확보한다. WNS 여유가 0.105 ns뿐이므로 기준선 없이는 어떤 변경도 평가할 수 없다.
2. **§F-1** — 4개 RTL 사본의 체크섬 검사 스크립트를 먼저 만든다. 그렇지 않으면 아래 변경들이 또 H-1과 같은 드리프트를 만든다.
3. 모든 변경은 `outputs/rtl_aead/rtl` → `zed_pqc/ip_repo/.../src` → Re-Package IP → Refresh Repositories → Generate Output Products(Global) → sim_1 fileset 재임포트 순서를 지킨다(OPTIMIZATION.md §"RTL 수정 후 Vivado 반영 절차").

### G-1. [High] 시뮬레이션/합성 사본 통일 — `mlkem_poly_tomsg_controller`

- **대상:** `outputs/rtl_aead/rtl/mlkem_poly_addsub_controller.sv:93-124` (및 4개 사본 전체)
- **변경:** `compress_1`을 32비트 상수 곱셈(`p=poly_rdata_i*32'd1290168+32'h40000000; message_o[index]<=p[31];`)에서 비교기 2개(`message_bit=(poly_rdata_i>=16'd833)&&(poly_rdata_i<=16'd2496);`)로 교체. 시뮬레이션 사본 D의 구현을 정본으로 채택.
- **검증:** G-2의 신규 TB로 `x ∈ [0,3328]` 전수 비교. 이후 `tb_mlkem512_decaps_shared_engine` / `tb_mlkem_secure_channel_complete_axi_top` 회귀.
- **리스크:** 조합 깊이 **감소** 방향. 32비트 곱셈 1개 제거로 DSP 1개 또는 약 40~60 LUT 절감 예상. **재합성으로 WNS 확인 필요.**

### G-2. [High] `mlkem_poly_tomsg_controller` 전용 TB 신설

- **대상:** `outputs/rtl_aead/tb/tb_mlkem_poly_tomsg_controller.sv` (신규)
- **변경:** 256계수를 0..3328 순회하며 소프트웨어 `compress_1` 참조값과 비교. 경계값 `0, 832, 833, 1664, 2496, 2497, 3328` 필수 포함. `$fatal`로 판정.
- **리스크:** 없음(TB 추가).

### G-3. [High] 슬롯 범위 검사를 하드웨어에 추가

- **대상:**
  - `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:241-242`
  - `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv:78`
  - `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:117-119`
- **변경:** 슬롯 레지스터 쓰기 시 `|wdata_q[31:SLOT_WIDTH]`를 검사해 범위 초과면 레지스터를 갱신하지 않고 `RESP_SLVERR` 반환(또는 sticky 오류 비트). `slot_in_range()`가 현 파라미터에서 항진명제임을 주석으로 명시하거나 슬롯 폭을 넓혀 실효화.
- **검증:** G-8의 프론트엔드 단위 TB에 슬롯 64/127 케이스 추가. **슬롯 0의 키가 보존되는지**를 반드시 확인(현재는 덮어써짐).
- **리스크:** 26비트 OR 리덕션(약 6 LUT). `s_axi_wdata → slot_reg` 경로는 크리티컬 아님. 낮음.

### G-4. [High] 인증 실패 시 태그 출력 차단

- **대상:**
  - `outputs/rtl_aead/rtl/aead_fixed64_engine.sv:150`
  - `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:390`
- **변경:** `tag_o <= poly_tag;` → 복호 방향에서는 기록하지 않음. `rsp_tag_o <= engine_tag_out;` → `rsp_tag_o <= (engine_auth_ok && !active_decrypt_q) ? engine_tag_out : 128'd0;`
- **검증:** 통합 TB에 "태그 변조 복호 후 `REG_OUTPUT_TAG`가 0인지" 확인 케이스 추가. 정상 암호화 경로의 태그는 그대로 나와야 함(`tb_mlkem_secure_channel_complete_axi_top.sv:427`의 `get_tag` 검사가 회귀 가드 역할).
- **리스크:** write-enable 게이팅으로 구현하면 0 LUT. 낮음.

### G-5. [Medium] 설치 후 키 재료 소거

- **대상:**
  - `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:339` (`REPORT` 상태)
  - `outputs/rtl_aead/rtl/mlkem_shared_hash_engine.sv:124` (`DONE` 상태)
  - `outputs/rtl_aead/rtl/secure_channel_material_bram_core.sv:65-68`
- **변경:** `REPORT`에서 `ss_a/ss_b`, `transcript_a/b`, `material_a/b`를 이중화 불변식(`x_a == ~x_b`)을 유지하며 0/전부1로 소거. `digest_o <= 0`, `material_q <= '0'` 추가.
- **검증:** TB에서 `dut.material_a`, `dut.ss_a`, `dut.hash.digest_o`가 `final_done` 이후 0인지 계층 참조로 확인.
- **리스크: 없음.** 신규 로직·조합 깊이 증가 없음. **가장 먼저 적용할 보안 개선 항목.**

### G-6. [Medium] Fault 시 세션 카운터 갱신 차단

- **대상:**
  - `outputs/rtl_aead/rtl/secure_channel_material_bram_core.sv` (포트 `fault_i` 추가)
  - `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:392`
  - `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:150-163` (`channel` 인스턴스 배선)
- **변경:** `if (engine_auth_ok)` → `if (engine_auth_ok && !fault_i)`. fault 시 카운터를 올리지 않고 `S_RESPONSE`로 직행.
- **검증:** fault 주입 중 복호 요청 후, fault 해제 뒤 같은 카운터로 정상 복호가 성공하는지 확인.
- **리스크:** AND 게이트 1개. 낮음.

### G-7. [Medium] 세션 무효화 / 소거 경로 신설

- **대상:**
  - `outputs/rtl_aead/rtl/secure_channel_material_bram_core.sv:80` (`cfg_valid_i(1'b1)`)
  - `outputs/rtl_aead/rtl/aead_session_manager_bram.sv:126-141, 261-277`
  - `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv` (`REG_REVOKE` 신설)
- **변경:** `revoke_i`/`revoke_slot_i` 포트를 추가해 `cfg_valid_i`를 실제 제어로 연결. 무효화 시 유효비트만 내리지 말고 `config_word()`가 `cfg_valid_q==0`일 때 항상 0을 반환하게 하여 24워드를 소거(기존 `S_CFG_WRITE` 재사용, 신규 상태 불필요).
- **선택 사항:** `session_valid`에 dual-rail 이중화 적용. **비교 트리가 새로 생기므로 `..._indexed_axi_top.sv:242-244`의 "비교 결과를 레지스터에 담고 다음 사이클에 판정" 패턴을 반드시 따를 것.** 그렇지 않으면 WNS를 잃을 수 있다.
- **검증:** revoke 후 해당 슬롯 트래픽이 `rsp_error_o`로 거부되는지, BRAM 24워드가 0인지 확인. `tb_aead_session_manager_bram.sv`에 케이스 추가.
- **리스크:** 기본 구현은 낮음. dual-rail 옵션은 **재합성 확인 필수.**

### G-8. [Medium] 프론트엔드 단위 TB 및 fault TB 이식

- **대상:**
  - `outputs/rtl_aead/tb/tb_aead_traffic_indexed_axi_lite_frontend.sv` (신규)
  - `outputs/rtl_aead/tb/tb_keccak_f1600.sv` (신규)
  - `outputs/rtl_aead/tb/tb_full_attack_fault_protection.sv:9` (DUT 교체)
- **변경:** fault TB의 DUT를 `mlkem_secure_channel_complete_axi_top`으로 교체하고 `request0`/`consume0`를 AXI 레지스터 접근으로 재작성(`tb_mlkem_secure_channel_complete_axi_top.sv`의 `aw`/`ar` 태스크 재사용). Keccak은 FIPS 202 KAT. 프론트엔드 TB는 슬롯 경계·미매핑 주소·중복 START를 커버.
- **리스크:** 없음(TB).

### G-9. [Medium] `mlkem_poly_accelerator` 제어 인덱스 폭 사이징

- **대상:** `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:44` 및 이를 쓰는 `:153-177, 213-348` 전체
- **변경:** `integer layer, span, block_start, butterfly, zeta_index, pair_index;` → `logic [2:0] layer; logic [9:0] span, block_start, butterfly; logic [7:0] zeta_index; logic [6:0] pair_index;`
- **주의:** `block_start+2*span >= 256` 비교가 최대 510에 도달하므로 **10비트 이상 필요**. 9비트로 잡으면 오버플로우로 무한 루프가 된다. `zeta_index<=(1<<(layer-1))-1`은 폭 고정 시프트 또는 `case(layer)`로 교체.
- **검증:** `tb_mlkem_poly_accelerator.sv`로 NTT/INTT/BaseMul 3개 커맨드 전수 재검증 **필수**. 이어서 `tb_mlkem512_decaps_shared_engine`, 통합 TB, `tb_mlkem_cycle_counter`(사이클 수 불변 확인).
- **리스크:** 약 148 FF / 180 LUT 절감, 32비트 캐리 체인이 10비트로 단축되어 **조합 깊이 감소 방향**. 다만 경계 조건 실수 시 기능이 깨지는 변경이므로 신중히. **재합성으로 WNS 확인 필요.**

### G-10. [Medium] 중복 레지스터 제거 (Keccak / Poly1305 / ChaCha20)

- **대상:**
  - `outputs/rtl_aead/rtl/keccak_f1600.sv:12, 119-126` — `state_o` 제거, `state_reg <= next_state`를 라운드 23에도 수행하고 `assign state_o = state_reg` (**1,600 FF**)
  - `outputs/rtl_aead/rtl/poly1305_fixed96.sv:37, 76, 151` — `message_reg` 제거, `current_block = message_i[128*block_index +: 128]` (**768 FF**)
  - `outputs/rtl_aead/rtl/aead_fixed64_engine.sv:41, 116, 132` — `poly_message_reg[767:640]` 상수 보관 제거 (**128 FF**)
  - `outputs/rtl_aead/rtl/chacha20_block.sv:15, 99-118, 126` — `initial_state`를 `always_comb` 재구성으로 대체 (**512 FF**)
- **합계:** 약 3,008 FF (전체 FF의 11.7%)
- **필수 부수 작업:** 각 모듈 헤더에 입력 안정성 계약을 명시한다. Keccak은 "`done_o` 어서션 사이클에 `state_o`를 캡처해야 하고 다음 `start_i`까지 최소 1사이클 간격이 필요"(현재 `sha3_shake_stream`이 유일 인스턴스이며 만족), ChaCha20/Poly1305는 "`busy_o` 동안 입력 포트 안정 유지 필요"(현재 `aead_fixed64_engine`이 만족). **계약 명시 없이 변경하면 재사용 시 사고가 난다.**
- **검증:** `tb_sha3_shake_stream`(SHA3-256/512, SHAKE128/256 KAT), G-8의 Keccak TB, `tb_aead_fixed64`(단 D-2의 `$fatal` 수정 선행), 통합 TB.
- **리스크:** 로직을 **제거**하는 변경이므로 타이밍은 중립~유리. 목적은 FF 확보가 아니라(FF 사용률 24%) **1600비트 레지스터 뱅크 제거로 Keccak 라운드 경로의 배치·라우팅 자유도를 높여 WNS 여유를 얻는 것.** 실제 효과는 **재합성 없이 확인 불가.**

### G-11. [Medium] Watchdog 우선순위 명시화 + 전체 연산 watchdog

- **대상:** `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:250-254, 278-341`
- **변경:** `if (timeout_trip) begin ... state <= REPORT; end else begin case(state) ... endcase end` 구조로 재작성해 watchdog이 실제로 최우선이 되게 한다. 상태별 `stage_timer`와 별도로 `total_timer`(20비트)를 추가한다.
- **검증:** 각 상태에서 강제 stall을 주입해 `F_TIMEOUT`이 실제로 `REPORT`로 보내는지 확인(현재는 6개 상태에서 덮어써짐).
- **리스크:** 재구성 자체는 로직 증가 없음. `total_timer`는 FF 20개 + 비교기 약 8 LUT이며 FSM 다음 상태 로직 팬인이 하나 늘므로 **재합성 확인 필요.**

### G-12. [Medium] AXI-Lite 견고성 — SLVERR, 메모리 창 보호, 읽기 경로 분할

- **대상:**
  - `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:97, 99, 110-146, 199-201, 249-256`
  - `outputs/rtl_aead/rtl/mlkem_decaps_axi_lite_frontend.sv:50, 87-93, 109`
  - `outputs/rtl_aead/rtl/mlkem_decaps_memory.sv:25`
- **변경:**
  1. 두 프론트엔드에 `decode_hit`를 만들어 미매핑/비정렬 주소에 `RESP_SLVERR` 반환 (§C-7)
  2. `REG_MEM_DATA` 접근을 `!decap_busy_i`로 게이팅해 복호 중 BRAM 쓰기 충돌 차단 (§B-6)
  3. `poly_mem[0:3071]` → `poly_mem[0:4095]`로 주소 공간과 일치시켜 범위 밖 접근 제거 (§C-6). BRAM 추론이 이미 4,096이므로 자원 증가 없음.
  4. 읽기 경로를 `araddr` 등록 + 범위 비교 기반 병렬 먹스로 재작성 (§C-8). LUT·깊이 양쪽에 유리하나 읽기 레이턴시가 1사이클 늘어나므로 통합 TB의 `ar` 태스크 가정 확인 필요.
- **리스크:** 1~3은 낮음. 4는 레이턴시 변화가 있어 TB 영향 확인 필요. 모두 **재합성 확인 권장.**

### G-13. [Medium, 선택] BaseMul / NTT / INTT 2사이클 파이프라이닝

- **대상:** `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:296-348`(BaseMul), `:213-240`(NTT), `:262-294`(INTT)
- **변경:** 스테이지를 겹쳐 연산당 2사이클로 단축. 각 스테이지의 곱셈 1개 구조는 유지(조합 깊이 불변). 레이어 경계에서 파이프라인 드레인.
- **기대 효과:** 약 36,700 사이클 절감 = 전체 105,286의 **약 35% → 약 686 µs.** PS `-O2`(1,258 µs) 대비 1.13배에서 약 1.8배로 개선.
- **전제 확인 완료:** 레이어 내 RAW 해저드 없음, `span >= 2`로 주소 충돌 없음, BaseMul은 쌍 간 의존성 없고 뱅크 분리. 완전 파이프라인(1사이클/연산)은 BRAM 4포트가 필요해 불가하므로 **2사이클이 상한.**
- **적용 순서(필수):** BaseMul → NTT → INTT. 매 단계마다 재합성해 WNS를 확인하고, 악화되면 즉시 중단.
- **리스크: 이 목록에서 가장 높음.** 동시 활성 DSP 수가 늘어 라우팅 혼잡과 DSP 입력 팬인이 변하므로 WNS +0.105 ns 설계에서 타이밍이 악화될 실질적 가능성이 있다. 추가 자원은 파이프라인 valid/인덱스 레지스터 약 100~200 FF. **검증 완료된 설계를 크게 건드리는 변경이므로, 대회 제출 일정에 여유가 없으면 G-1~G-12를 먼저 마치고 별도 브랜치에서 실험할 것을 권장한다.**

### G-14. [Low] 코드 위생

- `outputs/rtl_aead/rtl/aead_traffic_indexed_axi_lite_frontend.sv:148` — 동기 리셋을 도메인 관례인 비동기 리셋으로 통일 (§F-6)
- `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:82-115` — `zetas`를 `localparam` 배열로 바꾸고 두 읽기 인덱스를 먹스로 합쳐 ROM 이중화 제거(약 128 LUT) (§F-6)
- `outputs/rtl_aead/rtl/mlkem_shared_hash_engine.sv:112` — `out_index` 포화 시 `error_o` 어서트 (§F-6)
- `outputs/rtl_aead/rtl/mlkem_poly_accelerator.sv:156-160` — `span >= 2` 가정을 주석 또는 `assert`로 고정 (§E-8)
- `outputs/rtl_aead/rtl/mlkem_secure_channel_fault_protected_indexed_axi_top.sv:29` — `fault_inject_i`를 `parameter bit ENABLE_FAULT_INJECT`로 감싸 프로덕션 빌드에서 제거, `F_OUTPUT` 검출 조건을 주입 신호와 무관하게 재작성 (§F-5)

### G-15. RTL 변경이 아니지만 함께 처리해야 하는 항목

RTL 엔지니어의 작업 범위는 아니되 같은 변경 세트로 묶어야 하는 것들입니다.

| 항목 | 담당 | 참조 |
|---|---|---|
| `STATUS_KDF_BUSY` → `STATUS_FAULT` 개명, `REG_FAULT` 읽기 API 신설, `AEAD_HW_ERR_FAULT` 추가 | PS | §B-5 |
| 오류 반환 시 출력 버퍼 0 채우기 | PS | §D-4 |
| `Xil_Out32`/`Xil_In32` 또는 `dmb` 도입 | PS | §D-5 |
| `mlkem_decaps_hw_start`에 busy 폴링 추가 | PS | §B-6 |
| `session_generation` 비휘발화 또는 PL install-counter 도입(후자는 RTL + PC 동시 수정) | PS/RTL/PC | §B-4 |
| `pqc_64session_validation.c:88-92` 판정을 암호문 단독 비교로 강화 | PS | §D-3 |
| `tb_aead_fixed64.sv` 등 4개 TB에 `$fatal` 도입 | 검증 | §D-2 |
| XDC 확보, 4사본 체크섬 스크립트, `rtl/README.md`에 closure 표 작성 | 구현/문서 | §F-1, §F-2, §F-4 |
| README "범위 밖 슬롯 64 거부"를 "드라이버 레벨 거부"로 정정 | 문서 | §H-3 |
| `validate_ip.tcl` 절대 경로 제거 | 구현 | §F-3 |
| `pqc_session_scheduler` 제거 또는 배선, `first_set_bit(0)` 가드, `main` 중복 정리 | PS | §F-7 |

---

## 요약

| 우선순위 | 건수 | 항목 |
|---|---:|---|
| **High** | 5 | H-1 시뮬/합성 RTL 기능 드리프트, H-2 드리프트 모듈 TB 부재, H-3 HW 슬롯 범위 검사 부재, H-4 인증 실패 태그 노출, H-5 fault TB가 미합성 top 대상 |
| **Medium** | 19 | B-1~B-6 (보안 6), C-1~C-8 (RTL 구조·자원·타이밍 8), D-1~D-5 (검증·PS 5) |
| **Low** | 7 | F-1~F-7 (IP 패키징·저장소·PS 위생) |
| **검증 완료(문제 없음)** | 9 | E-1~E-9 |

가장 중요한 단일 발견은 **H-1**입니다. XSim이 컴파일하는 RTL과 비트스트림에 들어간 RTL이 `mlkem_poly_tomsg_controller`에서 기능적으로 다르고(양방향 드리프트), 해당 모듈에는 테스트벤치가 하나도 없습니다(H-2). 지금은 정규 계수 범위에서 두 구현이 등가이므로 동작하지만, "XSim 기능 검증 통과"가 합성된 설계에 대한 보증이 아니라는 상태 자체가 이 프로젝트의 모든 검증 결과에 걸린 전제를 약화시킵니다. README가 경고한 이중 사본 footgun은 실제로 **4중 사본**이며, 잘 관리된 축(`outputs/rtl_aead/rtl` ↔ 패키지 IP)이 아니라 관리되지 않는 축(Vivado `sim_1` 임포트 사본)에서 사고가 났습니다.
