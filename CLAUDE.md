# FPGA연구소 프로젝트

ZedBoard(Zynq-7000) 기반 ML-KEM-512 + ChaCha20-Poly1305 PQC 보안 채널 가속기를 대상으로, 기존 팀 저장소를 검토·개선하고 대회 제출 자료를 준비하는 프로젝트입니다.

**대상 저장소:** https://github.com/bobobomin/zedboard-pqc-secure
(clone 위치: `external/zedboard-pqc-secure/` — 용량 문제로 버전관리 대상에서 제외)

## 팀 구성 (커스텀 서브에이전트)

혼자 또는 소수 인원이 역할을 돌려가며 진행하며, 부족한 인력을 아래 서브에이전트로 보완합니다.

| 요청 내용 | 위임 대상 |
|---|---|
| 저장소/코드 검토, 개선점 찾기, 리뷰 보고서 | `fpga-code-reviewer` |
| RTL/로직/모듈 설계·구현 | `fpga-rtl-engineer` |
| testbench/검증/시뮬레이션 | `fpga-verification-engineer` |
| 합성/Vivado/타이밍/핀 제약/보드 디버깅 | `fpga-implementation-engineer` |
| 발표자료/보고서/제출 문서/데모 시나리오 | `fpga-doc-writer` |

역할이 겹치는 요청(예: "리뷰하고 개선된 RTL도 짜줘")은 관련 에이전트에 순차적으로 위임합니다.

**우선순위:** 이 저장소에서 코드 검토/RTL/검증/합성/문서 관련 요청은 항상 위 5개 `fpga-*` 에이전트를 우선 사용합니다. 전역 에이전트(`architecture-reviewer`, `security-specialist`, `technical-writer`, `test-writer` 등)와 `codebase-analyzer`/`security-audit` 등의 스킬은 이 프로젝트의 하드웨어 작업에는 사용하지 않습니다.

## 폴더 구조

- `external/` — 검토 대상 저장소 clone (버전관리 제외)
- `rtl/` — 이 프로젝트에서 작성/개선하는 RTL 코드 (공통 상수/파라미터는 `rtl/common/`)
- `tb/` — testbench / 시뮬레이션
- `constraints/` — 핀 배치, 타이밍 제약 파일 (.xdc)
- `scripts/` — 합성/구현 툴체인 실행 스크립트 (재현 가능하도록 커맨드를 남김)
- `docs/specs/` — 설계 문서
- `docs/review/` — 코드 검토 보고서
- `docs/` (그 외) — 대회 제출용 보고서/발표자료 초안
- `docs/superpowers/plans/` — 이 에이전트 팀 구성에 사용된 구현 플랜
- `references/` — 참고 자료 (논문, 규정집, 레퍼런스 설계 등). 자동 인덱싱은 없으며 필요 시 탐색해서 사용합니다.
- `.claude/agents/` — 이 프로젝트 전용 서브에이전트 5개
