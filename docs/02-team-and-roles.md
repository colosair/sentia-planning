# SENTIA 팀과 역할

> SENTIA 팀은 AI 2명, 백엔드 3명, 프런트엔드 1명으로 구성된다.
> 역할은 기술 스택이 아니라 각자가 책임지는 결과물을 기준으로 나눈다.
> 각 역할은 앞 역할의 결과물을 입력으로 받아 자기 결과물을 만들고, 그 결과물을 다음 역할에 넘긴다.
> 이 문서는 이후 시스템 구조와 일정·범위를 나누는 기준이 된다.

## 팀 구조

| 구분 | 역할 | 핵심 책임 |
|---|---|---|
| AI-1 | 모델·데이터 | 무엇이 보이는지 인지한다 |
| AI-2 | 파이프라인·Spatial AI | 관측을 시간·공간적으로 의미화하여 위험 후보를 만든다 |
| BE-1 | AI 연계·위험 도메인 | 위험 후보를 현장 업무 기준으로 판단하여 공식 사건으로 만든다 |
| BE-2 | 운영·이력 도메인 | 생성된 사건을 조치·검증·종료·기록까지 운영한다 |
| BE-3 | 플랫폼·실시간·인프라 | 서비스가 안정적으로 저장·전달·배포되도록 공통 기반을 책임진다 |
| FE-1 | 공간 관제 | 현재 현장과 사건을 보여주고 관리자 조작을 입력받는다 |

팀장은 AI 파트와 가장 가까운 백엔드 역할을 희망하며, 현재 구조에서는 BE-1 AI 연계·위험 도메인 역할과 가장 잘 맞는다.

이 문서에서 자주 쓰는 결과물 이름은 다음과 같다.

- `Detection`: AI 모델이 영상 한 장면에서 찾아낸 객체와 상태
- `Spatial State`: 작업자와 장비가 현재 현장의 어디에 있는지를 나타내는 상태
- `RiskCandidate`: AI가 위험해 보인다고 판단한 관측 후보
- `SafetyEvent`: 현장 업무 기준으로 처리하기로 결정한 공식 안전 사건

## 책임 흐름

```text
AI-1  모델·데이터
      인지 · 탐지
        │
        ▼  Detection
AI-2  파이프라인·Spatial AI
      추적 · 공간화 · 시간화 · VLM · 후보화
        │
        ▼  RiskCandidate          (Spatial State는 실시간 관제로 별도 전달)
BE-1  위험 도메인
      작업계획 · 위험성평가 · 현장 정책 적용
        │
        ▼  SafetyEvent
BE-2  운영 도메인
      검토 · 조치 · 검증 · 이력 · 보고
        │
        ▼  Domain State
BE-3  플랫폼
      API · 실시간 전달 · 공통 기반
        │
        ▼
FE-1  공간 관제
      관제 화면 · 관리자 조작
```

이 그림은 실제 네트워크 호출 순서를 뜻하지 않는다.
예를 들어 BE-2가 항상 BE-3를 호출한다는 의미가 아니며, 결과물의 책임이 어느 역할에서 어느 역할로 넘어가는지를 나타낸다.

## 역할 정의

각 역할은 같은 형식으로 설명한다.
비책임 영역은 그 역할이 하지 않는 일이며, 다른 역할의 핵심 책임과 겹치지 않도록 정한 경계다.

### AI 모델

`AI-1 · 모델·데이터`

> 현장 영상에서 필요한 객체와 상태를 인지할 수 있는 모델을 만든다.

| 항목 | 내용 |
|---|---|
| 입력 | 공개 데이터셋, 자체 촬영 데이터, Hard Negative, Annotation |
| 책임 영역 | 데이터 수집·정제, Annotation 기준, YOLO 학습, Fine-tuning, Detection, Segmentation, Augmentation, 모델 평가(Precision / Recall / mAP), Model Artifact 관리 |
| 출력 | Detection Model, Detection 결과 규격 |
| 기술 영역 | Python, PyTorch, YOLO, CVAT |
| 비책임 영역 | Tracking, Calibration, Homography, Pixel → Site 변환, RiskCandidate, SafetyEvent, 백엔드, 관제 UI |

### AI 파이프라인

`AI-2 · 파이프라인·Spatial AI`

> AI 모델의 Detection을 추적·좌표화·상태화하고, 필요한 AI 판단을 결합하여 `RiskCandidate`를 만든다.

| 항목 | 내용 |
|---|---|
| 입력 | CCTV / RTSP / Replay 영상, AI-1의 모델과 Detection, Camera Calibration 정보, 백엔드가 제공하는 현장 Context |
| 책임 영역 | Inference Runtime, Frame 처리, Tracking, Calibration, Homography, Pixel → Site Coordinate, Zone Membership 계산, Trajectory, Distance, Duration, Temporal State, Spatial State, Candidate Filtering, VLM Context Reasoning, AI Work Queue, RiskCandidate 생성 |
| 출력 | `Spatial State`: 실시간 관제용<br>`RiskCandidate`: 위험 판단용 |
| 기술 영역 | Python, FastAPI, OpenCV, ByteTrack, CUDA / GPU |
| 비책임 영역 | 작업계획 정본, 위험성평가 정본, 공식 Severity, SafetyEvent Lifecycle, 공통 DB 운영, FE Rendering, 공통 서비스 배포 |

두 출력은 목적이 다르기 때문에 서로 다른 경로로 처리한다.
VLM은 AI Work Queue를 거쳐 실시간 처리와 분리해서 실행하므로, VLM 응답이 늦어지더라도 Tracking과 `Spatial State` 갱신은 멈추지 않는다.
`Spatial State`의 계산 정본은 AI-2가 소유한다.
FE는 AI Runtime에 직접 연결하지 않고, BE-3의 실시간 전달 계층을 통해 `Spatial State`를 받는다.

AI-2는 AI 추론이 실제 서버에서 계속 동작하도록 하는 AI Runtime도 책임진다.
CUDA, GPU, AI Container, Model Loading, RTSP Runtime, Inference Health, FPS, Latency, VRAM이 이 범위에 들어간다.

### 위험 도메인

`BE-1 · AI 연계·위험 도메인`

> AI가 만든 `RiskCandidate`를 실제 현장 업무와 안전 정책에 연결하여 공식 `SafetyEvent`로 만든다.

| 항목 | 내용 |
|---|---|
| 입력 | RiskCandidate, 작업계획, 위험성평가, Site / Floor / Zone, 현장 정책 |
| 책임 영역 | Work Context, Risk Assessment Context, Zone Policy, Risk Policy, Candidate Validation, 업무 규칙, Severity 결정, 사건 중복 판단, SafetyEvent 생성 |
| 출력 | SafetyEvent |
| 기술 영역 | Java, Spring Boot, JPA, PostgreSQL / PostGIS |
| 비책임 영역 | Vision 알고리즘, Tracking, Calibration, SafetyEvent 생성 이후의 전체 Lifecycle, 공통 인프라, FE |

AI-2와 BE-1의 경계는 다음과 같다.

- **AI-2**는 위험해 보이는 관측 후보를 만든다.
- **BE-1**은 그 후보를 실제 업무 기준에서도 사건으로 처리할지 결정한다.

VLM은 BE-1이 실행하지 않는다.
BE-1은 `RiskCandidate`에 포함된 VLM 결과를 다른 AI 근거와 함께 참고하여 판단한다.

### 운영 도메인

`BE-2 · 운영·이력 도메인`

> 생성된 `SafetyEvent`를 실제 안전관리 업무 흐름에 따라 처리하고 종료한다.

| 항목 | 내용 |
|---|---|
| 입력 | SafetyEvent, 관리자 판단, 현장 조치 결과, Evidence |
| 책임 영역 | Event Lifecycle, Review, False Positive 처리, Action, Assignee, 작업자·현장 조치, ACK, Verification, Evidence Metadata, History, Timeline, Report, 위험성평가 피드백 |
| 출력 | 현재 Event State, Action State, History, Report Data |
| 기술 영역 | Java, Spring Boot, JPA, PostgreSQL |
| 비책임 영역 | AI 추론, RiskCandidate 생성, 위험 정책 자체, 실시간 인프라, 공통 배포, FE |

> **BE-1은 사건을 만들고, BE-2는 만들어진 사건을 처리하고 끝낸다.**

### 플랫폼

`BE-3 · 플랫폼·실시간·인프라`

> 다른 도메인이 안정적으로 실행되고 연결될 수 있도록 공통 서비스 기반을 책임진다.

| 항목 | 내용 |
|---|---|
| 입력 | 각 도메인의 API와 상태 변경, 배포 대상 서비스 |
| 책임 영역 | Auth, User, Permission, 공통 REST 기반, WebSocket / STOMP, Realtime Distribution, Redis, Notification, Object Storage 연동, 공통 Exception, Logging, Docker Compose, Nginx, Environment, CI/CD, Health Check, Monitoring, Deployment |
| 출력 | 공통 API 기반, 실시간 상태 전달, 배포된 서비스 환경 |
| 기술 영역 | Spring Boot, WebSocket / STOMP, Redis, Docker, Nginx |
| 비책임 영역 | CV, Tracking, Risk Policy, SafetyEvent 업무 규칙, Report 도메인, FE 화면 |

BE-3의 핵심은 플랫폼, 실시간 전달, 운영이라는 세 축이다.
BE-3는 다른 역할이 생성한 상태의 의미를 변경하지 않고, 이를 서비스 사용자에게 안정적으로 전달하는 기반을 책임진다.
연결 관리, 다중 클라이언트 전달, 재연결, 권한 적용이 이 범위에 들어가며, `Spatial State` 계산, `RiskCandidate` 생성, `SafetyEvent` 판단, Event Lifecycle 결정은 포함하지 않는다.
BE-3에게 제품 도메인 기능을 추가로 맡기지 않는다.

### 공간 관제

`FE-1 · 공간 관제`

> 백엔드의 정본 상태를 공간과 업무 흐름으로 보여주고, 관리자의 판단과 조작을 백엔드에 전달한다.

| 항목 | 내용 |
|---|---|
| 입력 | Spatial State, SafetyEvent, Action State, History, Evidence |
| 책임 영역 | 2D Floor Plan, 2.5D, 작업자·장비 위치, Zone, Camera, Event Queue, Event Detail, Evidence UI, Action UI, Timeline, Report UI, 실시간 상태 반영 |
| 출력 | 관제 화면, 관리자 입력 |
| 기술 영역 | React, TypeScript, Vite, Canvas, Three.js |
| 비책임 영역 | 거리 계산, Zone Membership 계산, 위험 판정, Severity 계산, 공식 상태 직접 변경 |

FE는 도메인 로직의 복사본을 가지지 않는다.
화면에 필요한 값은 백엔드와 AI가 계산한 결과를 받아서 표시하고, 관리자의 조작은 요청으로 백엔드에 전달한다.
FE가 받는 실시간 상태와 도메인 상태는 모두 백엔드 API와 BE-3의 실시간 계층을 통해 전달되며, AI Runtime에 직접 연결하는 경로는 기본 구조로 두지 않는다.

## 협업 경계

### 전달물

| 경계 | 전달물 | 보내는 쪽 | 받는 쪽 |
|---|---|---|---|
| 모델 경계 | Model Artifact / Detection | AI-1 | AI-2 |
| AI 경계 | RiskCandidate | AI-2 | BE-1 |
| 공간 경계 | Spatial State | AI-2 | BE-3 Realtime |
| 사건 경계 | SafetyEvent | BE-1 | BE-2 |
| 운영 경계 | Domain State | BE-2 | BE-3 |
| 서비스 경계 | Spatial State / Domain State / Event Update | BE-3 | FE |
| 사용자 경계 | Review / Action / Verification | FE | 백엔드 |

이 표는 네트워크 구현 순서가 아니라, 역할 사이에 주고받기로 약속한 결과물을 정리한 것이다.
구체적인 형식과 전달 방식은 `03-system-concept.md`와 실제 구현 문서에서 정한다.

주요 결과물이 실제로 전달되는 관계는 다음과 같다.
앞의 `책임 흐름` 그림이 역할의 논리적 순서를 나타낸다면, 이 그림은 결과물이 어느 역할로 전달되는지를 나타낸다.

```text
AI-1
  │ Detection
  ▼
AI-2
  ├─ RiskCandidate ──────→ BE-1
  │                          │ SafetyEvent
  │                          ▼
  │                        BE-2
  │                          │ Domain State
  │                          ▼
  └─ Spatial State ──────→ BE-3 Realtime
                             │
                             ▼
                            FE
```

BE-3는 `Spatial State`와 `Domain State`를 전달할 뿐이며, 두 상태를 계산하거나 의미를 바꾸지 않는다.

### 양방향 협업

AI와 백엔드 사이의 통신은 한 방향으로만 흐르지 않는다.

| 방향 | 전달 내용 |
|---|---|
| 백엔드 → AI | Site, Floor, Zone, Camera, Calibration Version, Active Work, Risk Assessment Context |
| AI → 백엔드 | Spatial State, RiskCandidate, Evidence Reference, AI Confidence, VLM Assessment |

작업 맥락을 **사용하는 것**과 **소유하는 것**은 구분한다.
작업계획, 위험성평가, 현장 구조 같은 업무·현장 정본은 백엔드가 소유한다.
AI 파이프라인은 백엔드가 제공한 Context를 읽어서 실시간 관측을 해석할 뿐, 그 정본을 따로 만들거나 수정하지 않는다.

## 인프라 경계

인프라는 별도의 7번째 역할로 만들지 않고, AI 런타임과 공통 플랫폼으로 나누어 맡는다.

### AI 런타임

주 담당은 **AI-2**다.

CUDA, GPU Runtime, AI Worker, Model Artifact Loading, RTSP, Inference Container, Inference Health, FPS, Latency, VRAM, AI Queue를 책임진다.

### 공통 플랫폼

주 담당은 **BE-3**다.

Docker Compose, Backend Runtime, PostgreSQL, Redis, Nginx, TLS, FE 배포, 공통 CI/CD, Environment, Secrets, Logging, Monitoring, Deployment, Object Storage를 책임진다.

두 영역이 만나는 지점에서는 다음과 같이 나눈다.

| 작업 | AI-2 | BE-3 |
|---|---|---|
| 지표 | AI Metric 정의 | Metric 수집·저장·대시보드 기반 |
| 컨테이너 | AI Dockerfile, CUDA Image | 전체 Compose, Network, Deploy |

## 기술 스택

기술 스택은 구인 공고와 기존 팀 방향을 따른다.
공통 기술축은 다음과 같다.

| 영역 | 기술축 |
|---|---|
| AI | Python, PyTorch, YOLO, OpenCV |
| 백엔드 | Java, Spring Boot, PostgreSQL, Redis |
| 프런트엔드 | React, Canvas, Three.js |

역할별 기술은 단계에 따라 다음과 같이 깊어진다.

| 역할 | MVP | 핵심제품 | 확장 |
|---|---|---|---|
| AI-1 | Python, PyTorch, YOLOv8/YOLO11, CVAT | Fine-tuning, Segmentation, 평가 자동화 | Pose, 추가 모델, 균열 모델 |
| AI-2 | Python, FastAPI, OpenCV, ByteTrack | Spatial/Temporal Engine, VLM, Redis/Celery, ONNX | BoT-SORT, ReID, TensorRT, 3DGS |
| BE-1 | Java, Spring Boot, JPA, PostgreSQL/PostGIS | Context/Risk Policy | 복합 Policy, 외부 연계 |
| BE-2 | Java, Spring Boot, JPA, PostgreSQL | Evidence, History, Report | 복합 Workflow, 외부 보고 |
| BE-3 | Spring Boot, WebSocket/STOMP, Redis, Docker, Nginx | Redis Streams, Object Storage, Monitoring, CI/CD | Kafka, Kubernetes, Multi-instance |
| FE-1 | React, TypeScript, Vite, Canvas, REST/STOMP | Three.js, 2.5D, 실시간 관제 | 3DGS, BIM, Unreal 연계 |

MVP, 핵심제품, 확장으로 넘어가는 과정은 기술을 교체하는 과정이 아니다.

```text
MVP       표준 기술로 전체 흐름을 처음부터 끝까지 연결한다
  ↓
핵심제품  같은 기술축을 더 깊게 사용한다
  ↓
확장      핵심 구조를 유지한 채 선택 기술을 추가한다
```

따라서 YOLO를 다른 Detector로 바꾸거나, Canvas를 폐기하거나, Spring Boot와 PostgreSQL을 다른 기술로 바꾸는 전환을 전제로 하지 않는다.

## 역할 부담

AI-2와 BE-3는 다른 역할보다 책임 범위가 넓다.
이것은 역할 분리가 잘못되었다는 뜻이 아니며, 두 역할 모두 단계별로 우선순위를 두어 부담을 나눈다.

| 단계 | AI-2 | BE-3 |
|---|---|---|
| MVP | Tracking, Calibration, Spatial State, RiskCandidate | Realtime, Docker, Redis, Nginx, 기본 배포 |
| 핵심제품 | Temporal 고도화, VLM, AI Queue | Monitoring, Object Storage, CI/CD, Redis Streams |
| 확장 | Multi-camera, TensorRT, 3DGS | Kafka, Kubernetes, Scale-out |

확장 단계의 항목은 선택 사항이며, 담당자가 반드시 구현해야 하는 필수 책임이 아니다.

## 책임 원칙

### 단일 정본

하나의 결과물은 하나의 역할만 정본으로 소유한다.

| 소유 역할 | 정본 |
|---|---|
| AI-1 | Detection: 인지 결과 |
| AI-2 | Spatial State, RiskCandidate: 공간·시간 관측과 AI 후보 |
| BE-1 | SafetyEvent: 공식 안전 사건 |
| BE-2 | Event State, Action State, History: 사건 처리 상태와 조치 이력 |
| BE-3 | 실시간 전달 상태와 플랫폼 운영 기반 |
| FE-1 | 화면 표현 상태 |

### 로직 중복

같은 위험 규칙을 AI, 백엔드, 프런트엔드에 각각 복제하지 않는다.

### 입출력 명확화

각 역할은 다음 역할에게 넘길 결과물을 명확하게 정의한다.

### 경계 우선

업무가 밀린다는 이유로 다른 역할의 핵심 책임을 임의로 가져오지 않는다.

### 확장 분리

3DGS, BIM, Unreal, Kafka, Kubernetes 같은 확장 기술은 현재 역할 경계를 깨지 않고 추가할 수 있어야 한다.

## 현재 상태

| 상태 | 내용 |
|---|---|
| 확정 | AI 2 / BE 3 / FE 1 구성<br>AI-1 = 모델·데이터<br>AI-2 = 파이프라인·Spatial AI<br>BE-1 = AI 연계·위험 도메인<br>BE-2 = 운영·이력 도메인<br>BE-3 = 플랫폼·실시간·공통 인프라 (Domain State와 Spatial State의 전달 기반이며, 소유자는 아님)<br>FE-1 = 공간 관제<br>AI 런타임 인프라 = AI-2<br>공통 인프라 = BE-3 |
| 논의 필요 | 실제 팀원별 최종 업무 배정<br>핵심 이벤트 선정에 따른 세부 책임량<br>VLM의 MVP·핵심제품 적용 시점<br>작업자 알림 Endpoint 범위<br>3DGS·Unreal 실제 담당 여부 |
| 확장 | Multi-camera<br>ReID<br>3DGS<br>BIM<br>Unreal<br>Kafka<br>Kubernetes |
