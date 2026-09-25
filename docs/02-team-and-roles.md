# SENTIA 팀과 역할

> SENTIA 팀은 AI 2명, 백엔드 3명, 프런트엔드 1명으로 구성한다.
> 역할은 기술 스택이 아니라 **어떤 결과의 정본을 누가 책임지는가**를 기준으로 나눈다.
> 이 문서는 01의 제품 방향과 03의 시스템 구조가 실제 팀 책임으로 어떻게 나뉘는지를 정의한다.

## 팀 구조

| 구분 | 역할 | 핵심 책임 |
|---|---|---|
| AI-1 | 모델·데이터 | 현실에서 무엇이 보이는지 인지하는 모델과 데이터를 책임진다 |
| AI-2 | Vision Pipeline·Spatial AI | Detection을 추적·공간·시간·맥락 정보와 결합해 Event Candidate와 AI 보강 결과로 만든다 |
| BE-1 | AI 연계·위험 판단 도메인 | 후보를 업무 맥락·정책과 결합해 Priority/Risk State와 관리자 검토 대상을 만든다 |
| BE-2 | 조치·검증·이력·보고 도메인 | 승인된 사건을 Action부터 Verify, Evidence, History, Report까지 운영한다 |
| BE-3 | 플랫폼·실시간·전달·인프라 | 실시간 상태와 알림을 안정적으로 전달하고 공통 플랫폼을 운영한다 |
| FE-1 | 공간 관제·업무 UI | 공간 관제, Event Queue, 관리자 검토, 조치·검증·보고 UI를 제공한다 |

팀장이 AI와 가장 가까운 백엔드 역할을 희망한 방향은 현재 구조에서 BE-1과 가장 잘 맞는다.

## 핵심 결과물

이 문서에서 사용하는 핵심 결과물은 다음과 같다.

| 결과물 | 의미 |
|---|---|
| Detection / Anomaly | AI 모델이 영상·이미지·센서 등에서 찾아낸 객체·상태·이상 |
| Spatial State | 관측 대상의 현장 위치와 공간 관계를 표현한 상태 |
| Event Candidate | 시스템이 추가 검토할 가치가 있다고 판단한 후보 |
| Event Queue | 후보를 Context·VLM·정책·관리자 판단으로 처리하는 논리적 검토 대기열 |
| Risk State / Priority | 후보의 공간·시간·작업 맥락을 반영한 검토용 위험 상태와 우선순위 |
| SafetyEvent | 관리자가 승인하여 실제 조치 대상으로 승격된 공식 안전 사건 |
| FalseAlarmRecord | 관리자가 기각한 후보의 판정 기록 |
| Action State | 승인된 사건에 대한 전달·ACK·조치·검증 상태 |
| Evidence / History | 조치와 판단의 증거 및 시간순 이력 |
| Report Draft | 구조화된 이력과 Evidence를 기반으로 AI가 생성한 보고서 초안 |

Event Queue는 DB나 Redis 같은 특정 구현을 뜻하지 않는다.
또한 Queue 자체가 정본이 아니다.
후보·상태·판정 기록이 정본이고 Queue는 이를 처리하기 위한 업무·실행 개념이다.

## 제품 책임 흐름

대표 안전 운영 흐름은 다음과 같다.

~~~text
AI-1
Detection / Anomaly
        │
        ▼
AI-2
Tracking / Spatial / Temporal
Event Candidate / VLM Assessment
        │
        ▼
BE-1
Event Queue
Context / Risk Policy
Priority / Risk State
        │
        ▼
관리자 검토
  ├─ 기각 ─→ FalseAlarmRecord
  │
  └─ 승인 ─→ SafetyEvent
                │
                ▼
BE-2
Action / Assignee
ACK / Verify
Evidence / History
Report Context
                │
        ┌───────┴────────┐
        ▼                ▼
BE-3 Delivery       AI-2 Report Worker
현장·관리자 전달       AI Report Draft
        │                │
        └───────┬────────┘
                ▼
FE-1
관제 / 검토 / 조치 / 보고서 검토
~~~

이 흐름은 네트워크 호출 순서를 뜻하지 않는다.
각 역할이 어떤 의미의 결과물을 책임지는지를 보여준다.

## 세 Pain Point와 역할

### 오탐·맥락 부족

단순 Detection을 바로 알람으로 보내지 않는다.

~~~text
Detection
→ Event Candidate
→ Event Queue
→ Spatial / Temporal / Work Context
→ 필요 시 VLM
→ Priority / Risk State
→ 관리자 승인 / 기각
~~~

역할은 다음처럼 나눈다.

- AI-1: 인지 품질
- AI-2: 추적·지속시간·공간 관계·후보화·VLM 보강
- BE-1: 업무 Context, 위험 정책, Priority/Risk State, 검토 Gate
- FE-1: 관리자의 승인·기각 UI
- BE-1: 승인 결과는 SafetyEvent, 기각 결과는 FalseAlarmRecord로 정본화

기각 이력을 Hard Negative나 Rule 개선에 활용할 수 있지만 자동 재학습을 기본 책임으로 두지는 않는다.

### 공간 정보 단절

- AI-2가 Pixel과 관측 결과를 Site/Floor/Zone 기준의 Spatial State로 변환한다.
- BE-3가 해당 상태를 실시간 전달한다.
- FE-1이 2D/2.5D 관제에 표현한다.
- Viewer는 정본이 아니며 동일 Spatial State를 소비한다.

### 조치·기록 단절

- BE-2가 승인된 SafetyEvent에서 Action을 만든다.
- BE-3가 관리자·현장 담당자 Endpoint로 전달한다.
- 현장 수신·ACK·조치·완료·Evidence는 BE-2의 Action State와 History로 돌아온다.
- BE-2가 구조화된 Report Context를 만들고 AI-2의 Report Worker를 호출할 수 있다.
- AI-2는 초안을 생성하고, BE-2가 Report Draft와 최종 검토 상태를 관리한다.
- FE-1에서 안전관리자가 초안을 검토·수정·확정한다.

스마트워치는 Notion에서 제시된 핵심 Target UX다.
다만 실제 자율 프로젝트에서 Native Watch 앱, 모바일, 웹, 모의 Endpoint 중 무엇으로 구현할지는 04에서 결정한다.
제품 구조에서는 **현장 전달과 응답 경로 자체를 제거하지 않는다.**

## 역할 정의

### AI-1 — 모델·데이터

> 프로젝트에서 필요한 현실 상태를 인지하기 위한 모델과 데이터를 책임진다.

| 항목 | 내용 |
|---|---|
| 입력 | 공개 데이터, 자체 데이터, Hard Negative, Annotation |
| 책임 | 데이터 수집·정제, Annotation 기준, Detection·Segmentation·필요 시 Anomaly/Pose/Temporal 모델, Fine-tuning, 평가, Model Artifact |
| 출력 | Model Artifact, Detection / Segmentation / Anomaly 결과 규격 |
| 기본 기술축 | Python, PyTorch, YOLO, CVAT 등 |
| 비책임 | Tracking, 좌표 변환, Event Queue, Risk State, SafetyEvent, 업무 Workflow, FE |

현재 대표 안전 흐름에서는 작업자·PPE·장비 Detection이 가장 구체적이다.
균열·구조 결함처럼 다른 관측 대상을 선택하면 Segmentation 등 다른 모델 계열을 사용할 수 있다.
구체 관측 대상은 04에서 정한다.

### AI-2 — Vision Pipeline·Spatial AI

> 모델이 인지한 현실을 추적·공간·시간 상태와 Event Candidate로 변환하고, 느린 AI 보강 작업을 실시간 경로와 분리하여 운영한다.

| 항목 | 내용 |
|---|---|
| 입력 | Camera/RTSP/Replay, Inspection Image, AI-1 결과, Calibration, Backend Context |
| 책임 | Inference Runtime, Tracking, Calibration, Homography, Pixel→Site, Zone Membership, Trajectory, Distance, Duration, Spatial/Temporal State, Candidate Filtering, Event Candidate, VLM Context, AI Work Queue |
| 추가 AI 책임 | 구조화된 Report Context를 입력으로 하는 AI Report Draft 생성 Worker |
| 출력 | Spatial State, Event Candidate, VLM Assessment, Report Draft |
| 기본 기술축 | Python, FastAPI, OpenCV, ByteTrack, PyTorch, CUDA/GPU |
| 비책임 | 작업계획·위험성평가 정본, 공식 Priority 정책, 승인/기각 정본, SafetyEvent Lifecycle, 최종 보고서 확정 |

AI-2가 모든 입력에 Tracking과 Homography를 적용하는 것은 아니다.

| 입력 계열 | 가능한 처리 |
|---|---|
| 동적 영상 | Detection → Tracking → Spatial/Temporal |
| Inspection | Detection/Segmentation → 결함 위치·구조 연결 |
| Sensor/Telematics | 시간 동기화·상태 결합 |
| Reality Capture | 공간 좌표·촬영 시점 연결 |

VLM은 모든 프레임을 처리하지 않는다.
Candidate 또는 검토가 필요한 상황을 별도 AI Work Queue에서 처리한다.

Report Draft 생성 역시 실시간 Tracking보다 낮은 우선순위의 Work Plane 작업이다.

### BE-1 — AI 연계·위험 판단 도메인

> Event Candidate를 현장 업무·정책과 결합해 관리자가 검토할 Risk State로 만들고, 승인·기각 결과를 공식 상태로 정리한다.

| 항목 | 내용 |
|---|---|
| 입력 | Event Candidate, VLM Assessment, Spatial State, 작업계획, 위험성평가, Site/Floor/Zone, 현장 정책 |
| 책임 | Work/Risk Context, Candidate Validation, Deduplication, Risk Policy, Priority, Risk State, Event Queue의 도메인 상태, 관리자 Review 결과 반영 |
| 승인 출력 | SafetyEvent |
| 기각 출력 | FalseAlarmRecord |
| 기본 기술축 | Java, Spring Boot, JPA, PostgreSQL/PostGIS |
| 비책임 | Vision 알고리즘, AI 모델 실행, 승인 이후 Action Lifecycle, 공통 전달 인프라, UI |

핵심 경계는 다음과 같다.

~~~text
AI-2
"검토할 가치가 있는 후보"

BE-1
"업무 맥락상 무엇을 우선 검토할지"

관리자
"실제로 조치할 사건인지"

승인
→ SafetyEvent

기각
→ FalseAlarmRecord
~~~

따라서 한 프레임의 Detection이 곧바로 SafetyEvent가 되지 않는다.

### BE-2 — 조치·검증·이력·보고 도메인

> 승인된 SafetyEvent를 실제 현장 조치와 검증, Evidence, 이력, 보고까지 운영한다.

| 항목 | 내용 |
|---|---|
| 입력 | SafetyEvent, 관리자 조치, 현장 ACK/결과, Evidence |
| 책임 | Action, Assignee, Delivery Request, ACK, Verification, Evidence Metadata, Event/Action History, Timeline, Report Context, Report Draft 상태, 관리자 검토·확정 기록, 위험성평가 Feedback Data |
| 출력 | Action State, Verification State, Evidence/History, Report Context, Report State |
| 기본 기술축 | Java, Spring Boot, JPA, PostgreSQL |
| 비책임 | Detection, Tracking, Event Candidate 생성, Risk Policy, AI Report 모델 실행, 공통 전달 인프라 |

BE-2는 보고서 텍스트를 임의 생성하는 역할이 아니라 **보고서의 업무 정본과 생성 입력을 책임진다.**

AI Report 흐름은 다음처럼 나눈다.

~~~text
BE-2
Event / Action / Evidence / History
        ↓
Report Context
        ↓
AI-2 Report Worker
        ↓
Report Draft
        ↓
BE-2
Draft / Review / Final 상태 관리
        ↓
FE-1
안전관리자 검토·수정
~~~

### BE-3 — 플랫폼·실시간·전달·인프라

> 다른 역할이 만든 상태와 Action을 사용자·현장 Endpoint에 안정적으로 전달하고 공통 실행 기반을 책임진다.

| 항목 | 내용 |
|---|---|
| 책임 | Auth, User, Permission, REST, WebSocket/STOMP, Realtime Distribution, Notification, Delivery Channel, Redis, Object Storage, Logging, Docker Compose, Nginx, CI/CD, Health, Monitoring, Deployment |
| 출력 | 공통 API, 실시간 전달, 알림·Action 전달 기반, 배포 환경 |
| 기본 기술축 | Spring Boot, Redis, WebSocket/STOMP, Docker, Nginx |
| 비책임 | Spatial State 계산, Event Candidate 판단, Risk Policy, Action 의미 결정, Report Domain |

BE-3는 의미를 만들지 않고 전달한다.

예:

~~~text
BE-2
"작업자 A에게 Action #17 전달"

        ↓

BE-3
Web / Mobile / Watch / Push 등 선택된 채널로 전달
~~~

실제 Channel은 04에서 결정한다.

### FE-1 — 공간 관제·업무 UI

> 안전관리자가 현실 상태와 Event Queue를 공간에서 이해하고, 승인·기각·조치·검증·보고 업무를 수행할 수 있는 UI를 책임진다.

| 항목 | 내용 |
|---|---|
| 입력 | Spatial State, Risk State, Event Queue, SafetyEvent, Action State, Evidence/History, Report Draft |
| 책임 | 2D Floor Plan, 2.5D, Zone/Camera/Object, Event Queue, Priority 표시, 승인·기각, Action UI, Verification, Evidence, Timeline, Report Draft 검토·수정 UI, 실시간 상태 반영 |
| 출력 | 관제 화면과 사용자 입력 |
| 기본 기술축 | React, TypeScript, Vite, Canvas, Three.js |
| 비책임 | 거리·Zone 계산, Risk 정책, 공식 상태를 클라이언트에서 임의 변경 |

제품 방향상 현장 작업자를 위한 가벼운 Endpoint도 필요하다.
Native Watch 앱을 FE-1이 직접 구현할지, 모바일/웹으로 검증할지는 04 범위 결정에 따른다.

## 실시간 Plane과 Work Plane

### Realtime Plane

~~~text
Camera
→ Detection
→ Tracking
→ Spatial State
→ BE-3 Realtime
→ FE
~~~

Spatial State의 계산 정본은 AI-2가 소유한다.
BE-3는 전달만 한다.

### Decision / Work Plane

~~~text
Event Candidate
→ Event Queue
→ Context / VLM / Spatial
→ Priority / Risk State
→ 관리자 Review
~~~

### Action / Report Plane

~~~text
SafetyEvent
→ Action Delivery
→ ACK
→ Verify
→ Evidence / History
→ Report Context
→ AI Report Draft
→ Human Review
~~~

느린 VLM이나 Report Generation 때문에 Realtime Plane이 막혀서는 안 된다.

## Queue 구분

Queue는 목적에 따라 구분한다.

### Event Queue

관리자가 검토해야 할 Event Candidate의 논리적 업무 Queue다.
BE-1이 후보의 상태, Priority, Review 결과 정본을 관리한다.

### AI Work Queue

VLM, 이미지 분석, Report Draft 같은 느린 AI 작업을 실행하기 위한 Queue다.
AI-2 Runtime이 처리한다.

### Delivery Queue

알림·Action을 사용자 Endpoint에 안정적으로 전달하기 위한 실행 구조다.
BE-3 영역이다.

구체적인 Redis/Celery/Streams/Kafka 선택은 구현 범위에서 정한다.

## AI ↔ Backend 양방향 Context

| 방향 | 내용 |
|---|---|
| Backend → AI | Site, Floor, Zone, Camera, Calibration Version, Active Work, Risk Assessment Context |
| AI → Backend | Spatial State, Event Candidate, Evidence Reference, Confidence, VLM Assessment, Report Draft |

업무 Context를 **사용하는 것**과 **소유하는 것**은 구분한다.
작업계획·위험성평가·현장 정책 정본은 Backend가 소유한다.

## 인프라 경계

인프라는 AI Runtime과 Common Platform으로 나눈다.

### AI Runtime — AI-2

- Inference Worker
- Edge AI Runtime 후보
- GPU/CUDA
- Model Loading
- Camera/RTSP Runtime
- VLM Worker
- Report Generation Worker
- AI Queue
- AI Health/FPS/Latency/VRAM

### Common Platform — BE-3

- Spring Runtime
- PostgreSQL/Redis
- Realtime/Notification
- Object Storage
- Docker Compose
- Nginx/TLS
- CI/CD
- Logging/Monitoring
- Deployment

AI-2가 AI Metric을 정의하고 BE-3가 공통 수집·대시보드 기반을 제공하는 식으로 협업할 수 있다.

## 기술 축

현재 기본 기술축은 유지한다.

| 영역 | 기본 기술축 |
|---|---|
| AI | Python, PyTorch, YOLO, OpenCV |
| Backend | Java, Spring Boot, PostgreSQL, Redis |
| Frontend | React, TypeScript, Canvas, Three.js |
| Runtime | Docker, Nginx, GPU/CUDA 필요 시 |

다음은 필요가 확인될 때 선택할 수 있는 기술이다.

- ByteTrack / BoT-SORT / ReID
- FastAPI
- Celery / Redis 계열 AI Queue
- ONNX / TensorRT
- PostGIS
- 3DGS / BIM / Unreal
- Kafka / Kubernetes
- Smartwatch Native Client

기술 목록이 제품 Scope를 자동으로 결정하지 않는다.

## 역할 부담 원칙

특정 기능을 추가하면 다음 역할의 부담이 증가할 수 있다.

| 범위 변화 | 영향 |
|---|---|
| 관측 대상·모델 추가 | AI-1 |
| Tracking·Spatial·VLM·Report AI 추가 | AI-2 |
| Risk Policy·Review 기준 확대 | BE-1 |
| Action·Evidence·Report 종류 확대 | BE-2 |
| Realtime·Notification·Endpoint 확대 | BE-3 |
| Viewer·Review·업무 화면 확대 | FE-1 |

역할을 무작정 넘기기보다 04에서 Scope를 줄이는 근거로 사용한다.

## 단일 정본

| 역할 | 정본 |
|---|---|
| AI-1 | Model Artifact와 Detection/Anomaly 규격 |
| AI-2 | Spatial State, Event Candidate의 AI 근거, VLM Assessment, AI Report Draft 결과 |
| BE-1 | Event Queue의 도메인 상태, Priority/Risk State, 승인·기각 결과, SafetyEvent/FalseAlarmRecord |
| BE-2 | Action State, Verification, Evidence Metadata, History, Report Workflow |
| BE-3 | 전달·연결 상태와 공통 플랫폼 운영 기반 |
| FE-1 | Presentation State |

AI Report Draft의 텍스트 생성 결과는 AI-2가 만들 수 있지만, 업무적으로 어떤 Draft가 현재 유효하고 누가 검토·확정했는지는 BE-2가 정본이다.

## 현재 상태

| 상태 | 내용 |
|---|---|
| 역할 방향 확정 | AI 2 / BE 3 / FE 1 구조<br>Detection과 Event Candidate 분리<br>관리자 승인·기각 Human Gate<br>승인 후 Action·Verify·Evidence·Report 연결<br>AI Runtime과 Common Platform 분리<br>Realtime Plane과 느린 Work Plane 분리 |
| 제품 방향상 포함 | Event Queue<br>False Alarm 분기<br>현장 Action Handoff<br>AI Report Draft + Human Review |
| 팀 합의 필요 | 핵심 Event 종류<br>Review UI 깊이<br>현장 Endpoint 구현 형태<br>보고서 종류·템플릿<br>VLM 사용 사건<br>2.5D 깊이 |
| 조건부 | 3DGS·BIM·Unreal<br>Multi-camera·ReID<br>Thermal·Wearable·Telematics<br>Kafka·Kubernetes |

구체적인 구현 범위와 일정은 04-scope-and-plan.md에서 결정한다.
