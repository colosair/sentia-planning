# SENTIA Documentation

SENTIA 문서는 현재 설계 정본과 과거 논의 기록을 분리하여 관리한다.

## Structure

- `00-discovery/`
  - 프로젝트 초기 조사, 세션별 논의, 가설과 방향 전환 기록
  - 현재 설계의 정본으로 사용하지 않는다.

- `01-product/`
  - 제품 정의, 사용자, 문제, Scope, MVP 등 제품 기획 정본

- `02-architecture/`
  - 전체 시스템 및 주요 파이프라인 아키텍처

- `03-contracts/`
  - Event, Observation, Spatial, API 등 구현 계약

- `04-data-and-validation/`
  - Dataset, 수집, Calibration, 검증 및 Demo Protocol

- `05-execution/`
  - 역할, Ownership, Roadmap, Milestone, Risk

- `06-extensions/`
  - Unreal, 3DGS, BIM 등 선택적 확장

- `adr/`
  - 주요 기술/제품 의사결정과 근거

## Document Authority

`00-discovery/`는 당시 세션과 사고 과정을 보존하기 위한 기록이다.

향후 현재 프로젝트의 공식 결정은
`01-product/`, `02-architecture/`, `03-contracts/` 등 정본 문서를 우선한다.

Discovery 문서와 정본 문서가 충돌할 경우 정본 문서를 따른다.
