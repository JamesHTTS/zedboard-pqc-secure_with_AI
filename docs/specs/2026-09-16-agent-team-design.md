# FPGA연구소 에이전트 팀 설계

## 배경
대회(하드웨어/임베디드, FPGA·디지털 회로 공모전) 프로젝트를 혼자·소수 인원이 역할을 돌려가며 진행 중이며, 부족한 인력을 Claude Code 커스텀 서브에이전트로 보완한다. 기존 팀 산출물은 GitHub 공개 저장소로 관리되며 로컬에 전체를 저장하기엔 용량 부담이 있어, 파일 첨부 대신 저장소 링크를 기반으로 검토한다.

대상 저장소: https://github.com/bobobomin/zedboard-pqc-secure
(ZedBoard/Zynq-7000 기반 ML-KEM-512 + ChaCha20-Poly1305 PQC 보안 채널 가속기. RTL, testbench, PS 드라이버, Vivado 2020.2 프로젝트, golden reference SW 구현으로 구성)

## 목표
1. 기존 저장소의 RTL/testbench/PS 드라이버를 검토하고 개선점을 도출
2. 개선점을 실제 RTL 코드로 반영
3. 검증(testbench)·합성/구현(Vivado)·발표자료 작성까지 이어지는 역할 분담 체계 마련

## 범위 밖
- 졸업작품(PIM, Processing-in-Memory) 팀 구성 — 별도 세션에서 계속 진행 예정, 이 스펙에 포함하지 않음
- 검토 대상 저장소에 대한 직접적인 코드 수정 PR (읽기 전용 검토만; 실제 반영은 이 프로젝트의 rtl/ 이하에서 진행)
- git 저장소 초기화 — 요청 시 별도 진행

## 폴더 구조
```
FPGA연구소/
├─ .claude/
│  └─ agents/
│     ├─ fpga-code-reviewer.md
│     ├─ fpga-rtl-engineer.md
│     ├─ fpga-verification-engineer.md
│     ├─ fpga-implementation-engineer.md
│     └─ fpga-doc-writer.md
├─ CLAUDE.md
├─ external/            # 검토 대상 저장소 clone 위치 (버전관리 제외)
├─ rtl/                 # 우리 프로젝트에서 새로/개선해 작성하는 RTL
├─ tb/                  # testbench / 시뮬레이션
├─ constraints/         # 핀 배치, 타이밍 제약 파일
├─ docs/
│  ├─ specs/            # 이 설계 문서 등
│  └─ review/           # fpga-code-reviewer가 작성하는 검토 보고서
├─ references/          # 참고 자료 (규정집, 논문, 레퍼런스 설계 등 — 단순 보관, 필요시 Grep/Read로 탐색)
└─ .gitignore           # external/ 등 제외
```

## 서브에이전트 정의

프로젝트 scope(`.claude/agents/`)로 등록 — james_telegram 등 다른 작업 디렉토리에는 노출되지 않음.

| 이름 | 담당 | 툴 | 모델 |
|---|---|---|---|
| `fpga-code-reviewer` | 대상 저장소 clone·검토, 코드 품질/타이밍/보안 로직(replay 방지, fail-closed, 세션 관리) 관점 개선점 도출, `docs/review/`에 보고서 작성 | Read, Write, Bash, Grep, Glob, WebFetch | sonnet |
| `fpga-rtl-engineer` | Verilog/VHDL RTL 핵심 로직 설계·구현 (리뷰 결과 반영 포함) | Read, Edit, Write, Bash, Grep, Glob | sonnet |
| `fpga-verification-engineer` | testbench 작성, 시뮬레이션 검증, 커버리지 확인 | Read, Edit, Write, Bash, Grep, Glob | sonnet |
| `fpga-implementation-engineer` | Vivado 등 툴체인 합성/구현, 타이밍·핀 제약, 보드 브링업 디버깅 | Read, Edit, Write, Bash, Grep, Glob | sonnet |
| `fpga-doc-writer` | 발표자료·보고서·데모 시나리오 작성 | Read, Edit, Write, Grep, Glob, WebSearch, WebFetch | sonnet |

각 에이전트 파일은 `~/.claude/agents/api-architect.md` 등 기존 전역 에이전트와 동일한 frontmatter 포맷(name/description(+`<example>` 트리거)/tools/model)을 따른다.

## `CLAUDE.md` 라우팅 규칙
- "저장소/코드 검토, 개선점 찾아줘, 리뷰해줘" → `fpga-code-reviewer`
- "RTL/로직/모듈 설계, 구현" → `fpga-rtl-engineer`
- "testbench/검증/시뮬레이션 결과" → `fpga-verification-engineer`
- "합성/Vivado/타이밍/핀 제약/보드에서 안 됨" → `fpga-implementation-engineer`
- "발표자료/보고서/제출 문서/데모 시나리오" → `fpga-doc-writer`
- 역할이 겹치는 요청(예: "리뷰하고 바로 개선된 RTL도 짜줘")은 순차로 여러 에이전트에 위임
- 프로젝트 개요, 대상 저장소 링크, 폴더 용도를 문서화

## 에러 처리 / 예외 상황
- 대상 저장소가 비공개로 전환되거나 접근 불가 시: `fpga-code-reviewer`가 clone 실패를 보고하고 중단 (인증 정보 요청하지 않음)
- `external/`은 매번 최신 상태로 재-clone하는 것을 기본으로 함 (별도 캐시 관리 없음 — YAGNI)

## 테스트/검증 방법
- 이 설계 자체는 설정 파일(마크다운 frontmatter + CLAUDE.md) 생성이므로 자동화 테스트 대상 아님
- 검증은 다음으로 확인: (1) 각 에이전트 파일이 올바른 frontmatter를 갖는지, (2) `fpga-code-reviewer`를 실제로 한 번 호출해 `external/`에 clone되고 `docs/review/`에 보고서가 생성되는지
