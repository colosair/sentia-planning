# SENTIA 시스템 개념

> SENTIA는 건설 현장의 현실 관측을 공간·시간·작업 맥락과 결합하여 Event Candidate를 만들고, 이를 관리자가 승인·기각한 뒤 실제 조치·검증·기록·AI 보고까지 연결하는 안전 운영 시스템이다.
>
> 이 문서는 제품이 어떤 정보 흐름으로 동작하는지를 설명한다. 이번 프로젝트에서 실제로 구현할 관측 대상과 기술 깊이는 04-scope-and-plan.md에서 팀 합의로 정한다.

## 시스템 개요

SENTIA의 핵심은 단순 Detection이 아니다.

~~~text
Sense
현실을 관측한다

        ↓

Understand
공간 · 시간 · 작업 Context로
실제로 검토할 상황을 선별한다

        ↓

Act
사람이 승인하고
현장 조치 · 검증 · 기록 · 보고로 연결한다
~~~

전체 제품 구조는 세 개의 운영 축으로 나뉜다.

1. **Realtime Plane** — 현재 현장 상태를 계속 갱신한다.
2. **Decision Plane** — 후보를 맥락화하고 관리자가 승인·기각한다.
3. **Action / Report Plane** — 승인된 사건을 현장 조치·검증·보고까지 연결한다.

이 세 축이 01의 세 Pain Point에 직접 대응한다.

| Pain Point | 시스템 대응 |
|---|---|
| 오탐·맥락 부족 | Event Candidate → Event Queue → Context/VLM/Spatial → Risk State → 승인/기각 |
| 공간 정보 단절 | Site/Floor/Zone 기반 Spatial State와 2D/2.5D 관제 |
| 조치·기록 단절 | Action Delivery → ACK/Verify → Evidence/History → AI Report Draft |

## 전체 흐름

~~~text
[업무 Context]
작업계획 · 위험성평가 · TBM
             │
             ▼
[Reality Input]
Camera / Sensor / Inspection / Replay
             │
             ▼
Detection / Segmentation / Anomaly
             │
       ┌─────┴───────────────┐
       │                     │
       ▼                     ▼
Realtime Plane         Decision Plane
Tracking              Event Candidate
Spatial State              ↓
       │                Event Queue
       │                    ↓
       │       Spatial / Temporal / Work Context
       │              + 필요 시 VLM
       │                    ↓
       │           Priority / Risk State
       │                    ↓
       │              관리자 Review
       │               ↙         ↘
       │            승인           기각
       │             ↓              ↓
       │         SafetyEvent    FalseAlarmRecord
       │             │
       │             ▼
       │        Action / Assignee
       │             │
       │             ▼
       │        현장 전달 / ACK
       │             │
       │             ▼
       │           Verify
       │             │
       │             ▼
       │      Evidence / History
       │             │
       │             ▼
       │        AI Report Draft
       │             │
       │             ▼
       │       관리자 검토·수정
       │             │
       │             ▼
       │    일지 / 보고 / Risk Feedback
       │
       └────────────→ 2D / 2.5D 관제
~~~

이 그림은 특정 API 호출 순서를 뜻하지 않는다.
개념적으로 어떤 정보가 어떤 판단을 거치는지 보여준다.

## 업무 맥락

다음 정보는 AI가 임의로 만드는 값이 아니라 현장 업무의 정본이다.

- 작업계획
- 공종과 작업 위치
- 인원과 장비
- 위험성평가
- TBM
- Site / Floor / Zone
- Camera와 Calibration Version

Backend가 이 정보를 정본으로 관리하고 AI Pipeline은 필요한 범위에서 읽어 관측을 해석한다.

같은 Detection이라도 다음에 따라 의미가 달라질 수 있다.

~~~text
어느 Zone인가
어떤 작업 중인가
몇 초 동안 지속됐는가
주변에 어떤 장비가 있는가
오늘 위험성평가에 포함된 위험인가
~~~

## 현실 입력

SENTIA는 CCTV 하나만을 전제로 하지 않는다.

| 입력 계열 | 예시 | 용도 |
|---|---|---|
| 영상 | CCTV, 녹화 영상 | 작업자·PPE·장비·행동·실시간 상태 |
| 이미지 | 점검 사진, 스마트폰, DSLR, 드론 | 균열·구조물 손상·근접 Inspection |
| 센서 | 환경 센서, Thermal, GPS/GNSS, Telematics, Wearable | 환경·장비·위치·행동 보조 |
| 공간 | 도면, CAD, BIM/IFC, 기준점 | Floor·Zone·공간 기준 |
| Reality Capture | 스마트폰 촬영, 360, Point Cloud, 3DGS | 주기적 현장 재현과 변화 |
| 가상 입력 | Replay, Simulation, Synthetic | 개발·검증·시연·희귀 사건 보강 |

이 목록은 Scope가 아니다.
어떤 입력을 실제 제품에 사용할지는 04에서 결정한다.

## Reality Interface

입력의 종류와 후속 처리 구조를 가능한 한 분리한다.

~~~text
Camera · Video · Image · Sensor · Replay · Simulation
                         │
                         ▼
                 Reality Interface
      수집 · 정규화 · 시간 동기화 · 검증 · 공간 연결
                         │
                         ▼
                 표준화된 관측 정보
~~~

CameraAdapter, ImageAdapter, SensorAdapter 같은 이름은 구현 강제가 아니라 개념 예시다.

Replay와 Testbed는 실제 공사 현장을 대체하는 임시 꼼수가 아니라 **동일 Pipeline을 반복 검증할 수 있게 하는 입력 전략**이다.

## 관측 처리 계열

현실 입력은 같은 방식으로 처리하지 않는다.

| 계열 | 대표 대상 | 개념 흐름 |
|---|---|---|
| 동적 관측 | 작업자, PPE, 장비, 위험구역 | Video → Detection → Tracking → Spatial/Temporal → Candidate |
| 구조 점검 | 균열, 박락, 누수, 구조 결함 | Inspection → Detection/Segmentation → Defect Analysis → Candidate |
| 환경 관측 | 화재, 연기, 과열, 가스 | RGB/Thermal/Sensor → State/Anomaly → Candidate |
| 현장 변화 | 공정, Reality 변화 | Periodic Capture → Reality Processing → Time Snapshot → Change |

어떤 관측 계열이 Core인지 현재 문서에서 결정하지 않는다.

## Hybrid Edge AI 방향

Notion에서 제시된 차세대 방향은 **Edge 기반 1차 실시간 감지 + VLM 기반 2차 상황 분석·문서화**다.

개념적으로는 다음처럼 분리할 수 있다.

~~~text
현장 또는 가까운 Runtime
Camera
→ 1차 Detection / Tracking
→ 즉시 Spatial State

              │ Candidate
              ▼

별도 Work Plane
Context / VLM
→ Risk Enrichment
→ Report Generation
~~~

목표는 다음 두 가지다.

- 실시간 경로의 Latency와 대역폭 부담을 낮춘다.
- 무거운 VLM·문서화 작업이 Tracking을 막지 않게 한다.

구체 Edge 장비와 배포 위치는 04의 기술 검증 대상이다.

## Realtime Plane

Realtime Plane은 **현재 현장 상태**를 유지한다.

~~~text
Camera
  ↓
Detection
  ↓
Tracking
  ↓
Spatial / Temporal State
  ↓
BE-3 Realtime
  ↓
FE
~~~

Realtime Plane의 정본은 Queue가 아니라 Spatial State다.

실시간 상태에는 필요에 따라 다음이 포함될 수 있다.

- 현재 작업자와 장비 위치
- Track
- Zone Membership
- 거리
- 속도
- 체류 시간
- PPE 상태
- Camera 상태

VLM, 관리자 Review, 보고서 생성이 느려져도 이 경로는 계속 동작해야 한다.

## Decision Plane

### Event Candidate

Detection 하나가 바로 알람이나 공식 사건이 되지 않는다.

~~~text
Detection / Anomaly
→ 지속시간 / 공간조건 / 중복 제거
→ Event Candidate
~~~

Event Candidate는 **추가 검토할 가치가 있는 관측 후보**다.

### Event Queue

Event Candidate는 Event Queue에서 검토 대상으로 관리된다.

~~~text
Event Candidate
        ↓
Event Queue
        ↓
Spatial State
Temporal State
Work Context
Risk Assessment
필요 시 VLM
        ↓
Priority / Risk State
~~~

Event Queue는 다음 의미다.

> 지금 당장 HIGH Alert를 발생시키는 목록이 아니라, 시스템과 관리자가 추가 판단해야 할 후보의 대기열

Event Queue가 특정 Redis List나 Kafka Topic이라는 뜻은 아니다.

### Priority / Risk State

후보는 다음 정보를 결합해 관리자에게 보여 줄 검토 상태로 정리된다.

- 공간 위치와 Zone
- 지속 시간
- 작업자·장비 관계
- 현재 공종
- 위험성평가
- AI Confidence
- 필요 시 VLM Assessment
- 동일 후보 반복 여부

정확한 Priority 계산식과 Severity 체계는 구현 범위에서 정한다.

### 관리자 승인·기각

Human-in-the-loop는 오탐 방지의 핵심 Product Gate다.

~~~text
Priority / Risk State
        ↓
    관리자 검토
      ↙       ↘
   승인         기각
    ↓            ↓
SafetyEvent   FalseAlarmRecord
~~~

기각된 후보는 현장 Action으로 전달하지 않는다.
따라서 단순 Detection과 실제 업무 Notification 사이에 **사람의 1차 승인 Gate**가 존재한다.

FalseAlarmRecord는 운영 통계와 품질 개선 근거로 남길 수 있다.
Hard Negative 활용이나 재학습 자동화 여부는 별도 결정이다.

## VLM

VLM은 모든 프레임을 판정하는 주 경로가 아니다.

~~~text
1차 Detection / State Filter
        ↓
Event Candidate
        ↓
필요한 후보만 VLM
        ↓
Context 보강 / 설명
~~~

VLM이 실패하거나 느려져도 Realtime Plane은 유지되어야 한다.

VLM의 실제 모델, 적용 사건, Timeout, Fallback은 04에서 검증한다.

## 공간 모델

공간 상태는 Viewer와 독립적으로 관리한다.

~~~text
Screen Space
     ↓
Drawing Space
     ↓
Site Local
     ↓
World / External Space
~~~

- Screen Space: 화면 표현용 좌표
- Drawing Space: 도면 위 좌표
- Site Local: 현장 기준 좌표
- World / External Space: 프로젝트 전역 또는 외부 공간 기준

World Space를 반드시 위경도나 특정 GIS 좌표로 정의하지 않는다.

정본은 Screen Pixel이 아니라 현장 공간 기준이다.

## Spatial State

다음 대상들이 같은 공간 기준을 공유할 수 있다.

- Site / Floor / Zone
- Camera
- Worker
- Equipment
- Event Candidate
- SafetyEvent
- Defect
- Sensor
- Reality Snapshot

이 구조 덕분에 Canvas, Three.js, Unreal이 서로 데이터를 변환하는 것이 아니라 동일한 Spatial State를 각각 표현할 수 있다.

## 시간 모델

SENTIA에는 여러 시간축이 존재한다.

### 실시간

~~~text
Frame
→ Track
→ 몇 초 지속되는 State
→ Event Candidate
→ Review
→ Action
~~~

### 업무

~~~text
작업계획
→ 작업 시간대
→ Action
→ Verification
→ 하루 이력
~~~

### 주기적 Inspection / Reality

~~~text
T0 → T1 → T2
~~~

정확한 저장 구조와 보존 기간은 이후 설계한다.

## Action / Report Plane

관리자가 승인한 SafetyEvent는 시스템 내부 기록으로만 끝나지 않는다.

### Action Delivery

~~~text
SafetyEvent
→ Action
→ Assignee
→ 현장 Endpoint 전달
→ ACK
→ 실제 조치
→ 완료 / Evidence
→ Verify
~~~

Notion에서는 안전관리자와 작업자의 스마트워치를 Target UX로 제시한다.
제품 개념에서는 **현장 전달·응답 경로가 존재한다는 것**이 핵심이고, 이번 프로젝트에서 Watch Native, Mobile, Web, Mock Endpoint 중 무엇을 사용할지는 04에서 결정한다.

### Evidence / History

Action 과정에서 다음을 구조화한다.

- 누가 판단했는가
- 누가 조치를 받았는가
- 언제 ACK 했는가
- 무엇을 조치했는가
- 언제 검증됐는가
- 어떤 이미지·영상·문서가 Evidence인가
- False Alarm은 몇 건이었는가

이 정보가 보고 자동화의 근거가 된다.

## AI 보고서 생성

보고는 업무 루프의 마지막 단계이며 단순 Report UI가 아니다.

~~~text
SafetyEvent
+ Action
+ Verification
+ Evidence
+ History
        ↓
Structured Report Context
        ↓
AI Report Worker
        ↓
Report Draft
        ↓
안전관리자 검토·수정
        ↓
업무 문서
~~~

활용 후보는 다음과 같다.

- 일일 안전일지
- 시정조치 이력
- 아차사고 기록
- 안전업무 보고
- 위험성평가 피드백

AI Report Draft는 최종 법적 판단이나 공식 결재를 대체하지 않는다.
사람이 검토하고 수정·확정하는 구조다.

Report Generation은 느린 Work Plane에서 실행되며 실시간 관제를 막지 않는다.

## Queue와 Backpressure

Discovery에서 유지할 중요한 원칙은 다음과 같다.

~~~text
Camera / Tracking
        >
Candidate Processing
        >
VLM Enrichment
        >
Report Generation
~~~

Queue가 쌓이는 것 자체가 즉시 장애는 아니다.

예:

- Report Queue가 늦어도 실시간 관제는 정상일 수 있다.
- Camera Frame Lag가 커지면 실시간 품질 문제다.

관측 후보:

- Queue depth
- oldest job age
- inference latency
- Camera lag
- GPU/VRAM
- Context/VLM latency
- Delivery 실패율
- Report backlog

동적 동시성이나 구체 Queue 기술은 구현 설계에서 결정한다.

## 저장 원칙

모든 AI 출력을 영구 저장할 필요는 없다.

| 대상 | 방향 |
|---|---|
| 원본 Frame/Detection | 단기 또는 선택 저장 |
| Spatial State | 최신 상태 중심 |
| Event Candidate | Review에 필요한 범위 저장 |
| SafetyEvent | 공식 사건 정본 |
| FalseAlarmRecord | Review 결과와 품질 통계 근거 |
| Action/Verification | 업무 이력 |
| Evidence | 사건·조치 근거 |
| Report Draft/Final | 업무 문서 상태 |

정확한 보존 기간과 Object Storage 구조는 이후 정한다.

## 표현 계층

같은 상태를 여러 Viewer가 표현할 수 있다.

~~~text
Spatial / Domain State
        │
        ├── Canvas / SVG      2D
        ├── Three.js          2.5D
        ├── 3DGS              Reality Layer
        ├── BIM / Mesh        Geometry / Semantics
        └── Unreal            Immersive / Simulation
~~~

2D/2.5D는 현장 운영에서 빠르게 위치와 상황을 파악하기 위한 기본 방향이다.

3DGS는 실제 시공 현장의 시각적 Reality를 주기적으로 연결하는 방향이며 CCTV의 실시간 동적 관측을 대체하지 않는다.

~~~text
고정 CCTV
→ 지속적인 현재 상태

Periodic Reality Capture
→ 시점별 현장 Snapshot
~~~

BIM/Mesh는 구조·의미·기하 정보, 3DGS는 실제 모습, Unreal은 통합 시각화·Simulation 역할로 분리할 수 있다.

실제 구현 여부는 04에서 결정한다.

## 현재 상태

| 상태 | 내용 |
|---|---|
| 방향 확정 | Realtime / Decision / Action-Report Plane 분리<br>Detection과 공식 사건 분리<br>Event Queue 기반 추가 판단<br>관리자 승인·기각 Human Gate<br>Spatial State를 현장 좌표 기준으로 관리<br>승인 후 현장 Action Handoff·ACK·Verify 연결<br>History/Evidence 기반 AI Report Draft<br>최종 판단과 보고 확정은 사람 |
| 팀 합의 필요 | 핵심 관측·사건<br>Priority/Risk State 세부 기준<br>현장 Endpoint 구현 형태<br>보고서 종류·형식<br>VLM 실제 적용 범위<br>2D/2.5D 구현 깊이 |
| 검증 필요 | 모델·오탐 성능<br>Tracking/Calibration<br>Queue latency/backpressure<br>현장 전달 신뢰성<br>AI Report Draft 품질<br>Replay/Testbed/실제 데이터 |
| 조건부 확장 | 3DGS<br>BIM 고도화<br>Unreal<br>Thermal/Telematics/Wearable<br>Multi-camera |

역할별 정본과 책임은 02-team-and-roles.md를 따른다.
