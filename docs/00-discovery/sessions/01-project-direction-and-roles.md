# SENTIA 자율 프로젝트 세션 종합 정리
> 작성 기준: 2026-09-25  
> 목적: 본 세션에서 논의된 프로젝트 방향, 팀 구성, 역할 경계, 기술 아키텍처, 실무 조사, 도메인 지식 활용 범위 및 ASC 적용 가능성까지 빠짐없이 통합 정리

---

## 0. 세션 핵심 결론

이번 프로젝트는 단순한 `YOLO + 웹 대시보드`가 아니라 다음 흐름을 목표로 하는 것으로 정리되었다.

```text
Reality / CCTV
   ↓
Perception AI
   ↓
Tracking / Spatial Mapping
   ↓
Context / State / Risk Reasoning
   ↓
Incident
   ↓
2D/2.5D Spatial Twin
   ↓
Action / Report Automation
```

프로젝트의 핵심 문제의식은 다음 3개로 수렴했다.

1. **무겁고 깨지기 쉬운 3D/BIM 중심 관제의 한계**
2. **단순 객체 탐지의 오탐 및 맥락 부재**
3. **감지 이후 실제 조치·보고 업무와의 단절**

AI Pipeline / Spatial Vision 담당의 주 포지션은 최종적으로 다음처럼 정리되었다.

> **AI/Vision 파트 — Vision Pipeline / Spatial Vision / Spatial AI**

즉, 모델을 직접 만드는 역할보다 **모델이 본 영상을 Tracking·좌표 변환·공간 상태로 바꾸고 Backend/Digital Twin으로 연결하는 역할**이 핵심이다.

---

# 1. 최초 프로젝트 구조 이해

초기에는 다음과 같은 전체 흐름으로 이해했다.

```text
[카메라 / CCTV]
      ↓
[AI 분석]
      ↓
Observation JSON
      ↓
[Backend]
- 상태 관리
- 위험 판정
- 로그/이력 저장
      ↓
WebSocket / SSE
      ↓
[Frontend]
- 2D / 2.5D 공간 표현
- 알림 / Incident / Timeline
```

중요한 보정은 다음과 같다.

```text
AI Observation
      ↓
Backend Interpretation
      ↓
Twin State
      ↓
Frontend Representation
```

즉 단순히 `AI JSON → BE → FE 중계`가 아니라 각 계층이 한 단계씩 의미를 부여한다.

### 역할별 의미

- **AI**
  - "무엇이 보였는가?"
  - "누구인가?"
  - "현실 공간 어디에 있는가?"
- **Backend**
  - "그 관측이 어떤 상태인가?"
  - "위험인가?"
  - "Incident로 기록해야 하는가?"
- **Frontend**
  - "사용자에게 어떻게 표현할 것인가?"

---

# 2. 전체 데이터 흐름

## 2.1 물리 세계에서 화면까지

```text
현실 현장
   ↓
Camera / RTSP
   ↓
Frame decode / sampling
   ↓
Detection
   ↓
Tracking
   ↓
Camera Calibration / Homography
   ↓
Pixel → World / Floorplan Coordinate
   ↓
Spatial Observation
   ↓
Async AI → Backend
   ↓
Backend Twin State
   ↓
Risk / Context / Incident
   ↓
DB / History
   ↓
WebSocket / SSE
   ↓
Frontend Twin Store
   ↓
World → Screen
   ↓
2D / 2.5D Rendering
```

## 2.2 AI 파트의 책임 경계

AI/Vision 파트는 기본적으로 다음 구간을 소유한다.

```text
Raw Video
   ↓
Detection
   ↓
Tracked Object
   ↓
Spatial Object
   ↓
Observation
```

Backend는 그 다음을 소유한다.

```text
Observation
   ↓
Domain State
   ↓
Risk
   ↓
Incident
```

Frontend는 다음을 소유한다.

```text
Twin State
   ↓
Visual State
   ↓
2D / 2.5D Representation
```

---

# 3. Vision Pipeline / Spatial Vision 세부 책임

## 3.1 영상 입력

- RTSP / CCTV / 녹화 영상
- frame decode
- resize
- sampling
- inference FPS 조절
- buffer 관리
- frame drop
- reconnect
- latency 관리

예시:

```text
Camera 30 FPS
   ↓
AI inference 5~10 FPS
   ↓
Tracking
   ↓
Spatial Observation
```

모든 프레임 결과를 Backend에 보낼 필요는 없다.

---

## 3.2 Detection

모델 출력 예시:

```json
[
  {
    "class": "person",
    "bbox": [710, 340, 820, 690],
    "confidence": 0.96
  },
  {
    "class": "excavator",
    "bbox": [1020, 300, 1510, 760],
    "confidence": 0.91
  }
]
```

Detection은 기본적으로 다음 수준이다.

> "이 프레임의 이 위치에 작업자가 있다."

아직 지속 객체가 아니다.

---

## 3.3 Tracking

Detection을 시간축으로 연결한다.

```text
Frame 100 → Worker #17
Frame 101 → Worker #17
Frame 102 → Worker #17
```

Tracking의 핵심 이슈:

- track ID
- object lost
- reappeared
- ID switch
- occlusion
- temporal smoothing

후보:

- ByteTrack
- BoT-SORT
- DeepSORT 계열

---

## 3.4 Pixel → World / Floorplan 좌표 변환

이 프로젝트의 핵심 기술 중 하나다.

```text
영상 좌표
(x = 785 px, y = 690 px)

      ↓
Calibration / Homography

현장 좌표
(x = 13.4 m, y = 7.8 m)
```

주요 학습 대상:

- OpenCV
- Camera Calibration
- Homography
- Perspective Transform
- Intrinsic / Extrinsic
- Origin / Scale / Rotation
- Floorplan coordinate
- Geometry

이 부분은 단순 부가 기능이 아니라 **경량 Spatial Twin의 핵심 차별화 레이어**다.

---

## 3.5 Observation 생성

AI 최종 산출물 예시:

```json
{
  "cameraId": "CAM-03",
  "timestamp": "2026-09-23T10:31:24.231Z",
  "objects": [
    {
      "trackId": 17,
      "type": "WORKER",
      "confidence": 0.95,
      "position": {
        "x": 13.4,
        "y": 7.8
      },
      "attributes": {
        "helmet": false
      }
    }
  ]
}
```

AI는 여기까지 말한다.

> Worker #17이 (13.4, 7.8)에 있고 안전모가 탐지되지 않았다.

Backend가 그 다음을 판단한다.

> 이 작업자는 안전모 필수구역 안에 있으므로 PPE 위반이다.

---

## 3.6 비동기 전송

좋은 구조:

```text
Inference Worker
      ↓
Observation Queue
      ↓
Async Sender
      ↓
Backend
```

나쁜 구조:

```text
Frame
 ↓
AI
 ↓
POST Backend
 ↓
응답 대기
 ↓
다음 Frame
```

MVP에서는 `async HTTP` 정도로 충분하며, 필요해질 경우:

- Redis Streams
- RabbitMQ
- Kafka

등으로 확장 가능하다.

---

# 4. Backend 역할

Backend는 단순 중계 서버가 아니다.

## 4.1 공식 Twin State 관리

예:

```json
{
  "entityId": "worker-17",
  "type": "WORKER",
  "position": {
    "x": 13.4,
    "y": 7.8
  },
  "zoneId": "ZONE-A",
  "status": "ACTIVE",
  "risk": {
    "type": "PPE_VIOLATION",
    "severity": "HIGH"
  }
}
```

시스템의 정본은 Backend에 두는 것이 자연스럽다.

---

## 4.2 Risk Engine

예:

```text
Worker #17
position = (13.4, 7.8)
helmet = false

+

Zone A
helmetRequired = true

↓

Worker #17 ∈ Zone A
AND helmet == false

↓

PPE_VIOLATION
```

즉:

- AI = 관측
- Backend = 해석/정책
- Frontend = 표현

---

## 4.3 저장 정책

AI 결과를 전부 DB에 저장하면 데이터가 폭증한다.

구분:

```text
Raw Detection       → 대부분 폐기
Current State       → 유지
Important Event     → 영구 저장
History Snapshot    → 필요 주기로 저장
```

예:

```text
10:31:24 PPE_VIOLATION CREATED
10:31:31 PPE_VIOLATION RESOLVED
```

---

# 5. Frontend 역할

## 5.1 World → Screen 변환

Backend:

```text
(13.4m, 7.8m)
```

Frontend:

```text
(412px, 291px)
```

즉:

- Camera Pixel → World = Vision/Spatial
- World → Screen = Frontend

---

## 5.2 2D / 2.5D 표현

2D:

```text
┌────────────────────────┐
│           🚜           │
│                        │
│      🔴17              │
│                        │
└────────────────────────┘
```

2.5D:

```text
        ╱────────────╱│
       ╱            ╱ │
      │     👷17    │ │
      │             │╱
      └─────────────┘
```

Backend 데이터 구조는 렌더링 방식에 종속되면 안 된다.

---

# 6. AI가 2명일 때 역할 분리

가장 자연스러운 분리는 다음이다.

## AI-1 — Model / Data

질문:

> "무엇이 보이는가?"

책임:

- Dataset
- Annotation
- YOLO Fine-tuning
- Segmentation
- Augmentation
- Precision / Recall / mAP
- 모델 최적화
- threshold
- 모델 artifact

최종 출력:

```json
{
  "class": "WORKER",
  "bbox": [710, 340, 820, 690],
  "confidence": 0.96
}
```

---

## AI-2 — Vision Systems / Spatial AI

질문:

> "그 객체가 누구이며 현실 어디에 있는가?"

책임:

- Video / RTSP ingestion
- model inference integration
- Tracking
- Calibration
- Homography
- Pixel → World
- Spatial smoothing
- Observation 생성
- Async AI → BE
- AI Runtime
- GPU runtime
- health / readiness / metrics

최종 출력:

```json
{
  "cameraId": "CAM-03",
  "trackId": 17,
  "type": "WORKER",
  "position": {
    "x": 13.4,
    "y": 7.8
  }
}
```

이 역할이 AI Pipeline / Spatial Vision 담당의 주 포지션이다.

---

# 7. Helmet 같은 속성의 경계

예:

```text
AI-1
"helmet이 보인다 / 안 보인다"

AI-2
"이 helmet 관측이 Worker #17에 속한다"

Backend
"Worker #17이 안전모 필수구역에서 helmet=false이므로 위반이다"
```

정리:

```text
AI-1        AI-2             Backend

인식         객체화/공간화       의미/정책화
무엇인가? → 누구/어디인가? → 그래서 위험한가?
```

---

# 8. AI 2명이 정말 필요한가?

## 8.1 1명으로 충분한 경우

- 카메라 1~2대
- 사전학습 YOLO
- 소규모 Fine-tuning
- 간단 Tracking
- 고급 좌표 보정 없음
- Segmentation 없음
- 실시간 시스템 복잡도 낮음

이 경우 AI 1명도 가능하다.

---

## 8.2 2명이 적절한 경우

다음을 모두 하려면 2명이 합리적이다.

```text
Fine-tuning
+
Realtime Video Inference
+
Tracking
+
Spatial Mapping
+
Digital Twin
```

특히:

```text
모델 성능 자체가 핵심
+
실시간 Twin 연동도 핵심
```

이면 두 역할은 사실상 다른 직무다.

---

# 9. 프로젝트 인프라

초기에는 이전 Unity WebGL 기반 프로젝트보다 훨씬 가벼울 가능성이 높다고 판단했다.

## 9.1 이전 Unity WebGL 기반 프로젝트와 비교

이전 프로젝트:

```text
React
Unity WebGL
Game Runtime
Spring
Realtime
Cloudflare
Large Artifacts
Jenkins
demo/prod
CI/CD
```

이번 프로젝트:

```text
React
Spring
PostgreSQL
Redis(optional)
AI Worker
GPU
RTSP
Docker Compose
```

차이:

- Unity WebGL 없음
- 대형 WebGL artifact 없음
- CDN/cache 복잡도 낮음
- 기존 Jenkins 기술부채 없음
- Greenfield 가능성 높음

---

## 9.2 대신 새로 생기는 Infra 특수성

- NVIDIA Driver
- CUDA
- PyTorch
- GPU Container Runtime
- RTSP
- AI Worker lifecycle
- model artifact
- inference health
- FPS / latency metrics

즉:

> 웹 인프라의 복잡도는 낮아지고, AI/GPU Runtime 특수성이 커진다.

---

# 10. Vision Pipeline + Infra를 함께 맡을 수 있는가?

가능하다고 정리했다.

특히 프로젝트 전체가 다음 수준이라면:

```text
1 GPU Server

docker compose
├ frontend
├ backend
├ ai-worker
├ postgres
└ redis(optional)
```

Vision Systems 담당이 아래까지 가져가는 것은 자연스럽다.

```text
Vision Systems
├ RTSP ingest
├ inference integration
├ tracking
├ calibration
├ coordinate mapping
├ async observation pipeline
│
└ Runtime / Infra
   ├ Docker
   ├ CUDA / GPU
   ├ model artifact
   ├ health / restart
   ├ CI/CD
   └ logs
```

다만 다음은 MVP에서 피한다.

- Kubernetes
- Kafka Cluster
- multi-node GPU scheduling
- 복잡한 Observability Stack
- HA DB
- Blue/Green Production
- 복잡한 artifact promotion

핵심은:

> **Infra를 별도 잡무로 추가하는 게 아니라, Vision System을 실제 서버에서 안정적으로 돌리기 위한 Runtime 책임으로 묶는다.**

---

# 11. AI Pipeline / Spatial Vision 담당의 주 학습 영역

이번 프로젝트에서 해당 역할이 주력으로 가져갈 영역:

## 11.1 Computer Vision 실전

- OpenCV
- bbox / mask / confidence 해석
- Detection 후처리
- Tracking
- ID switch / occlusion / lost
- temporal smoothing

## 11.2 Spatial / Geometry

- Camera Calibration
- Homography
- Perspective Transform
- Pixel → World / Floorplan
- 좌표계
- polygon / 거리 연산

## 11.3 Realtime Vision Pipeline

- RTSP
- frame sampling
- queue
- backpressure
- frame drop
- latency
- FPS
- async processing

## 11.4 AI Runtime

- PyTorch / ONNX
- CUDA
- GPU inference
- containerized worker
- model artifact / version
- health / readiness
- observability

## 11.5 Digital Twin Integration

- Detection → Tracked Entity → Spatial Observation
- AI → BE Contract
- FE 좌표 정합
- end-to-end sync validation

---

# 12. 모델링 영역은 별도 담당

이미 모델 학습 담당을 별도로 구한 상황이므로, AI Pipeline 담당은 모델링을 주 책임으로 가져갈 필요가 없다.

모델 담당의 주 영역:

- Dataset
- Annotation
- YOLO Fine-tuning
- Segmentation
- Augmentation
- Precision / Recall / mAP
- Hyperparameter tuning

AI Pipeline 담당은 다음 수준까지 이해하는 것이 적절하다.

- confidence 의미
- NMS
- false positive / false negative
- recall
- model FPS
- model artifact
- 모델 실패 패턴

즉:

> **모델을 연구하는 사람은 아니지만, 모델의 특성과 실패 양상을 이해하고 실제 시스템에서 안정적으로 사용하는 사람**

을 목표로 한다.

---

# 13. 역할 합의 결과

AI Pipeline / Spatial Vision 담당자는 프로젝트 합류 시점부터 AI Pipeline 역할을 희망했으며, 팀장과의 협의를 통해 AI Pipeline 및 Spatial Vision 영역을 최우선으로 배치하는 것으로 정리되었다. 추가 팀 빌딩은 팀원 소개를 통해 열어두었다.

이로 인해:

> **AI Pipeline / Spatial Vision 담당으로 합류 사실상 확정**

이라고 정리했다.

---

# 14. 팀장 역할 분석

팀 내에 건설 현장 실무 경험을 가진 도메인 담당자(팀장)가 존재한다. 다만 최신 3D/BIM 관제 솔루션을 실무에서 직접 운용한 경험과는 구분해야 하므로, 현장 경험은 문제 가설 수립에 활용하고 최신 제품·시장 현황은 별도 조사한다.

따라서 예상 역할은:

```text
Team Lead / PM        매우 높음
Domain Owner          매우 높음
Backend               높음
Infra / Deployment    중~높음
AI Application        중간
CV Model              낮음
```

---

# 15. 도메인 지식 활용 범위

중요한 보정:

팀 내 도메인 담당은 건설 현장 실무 경험이 있지만, **최근 도입되는 3D/BIM 관제 솔루션을 실제로 사용한 경험은 없음**.

따라서 다음처럼 분리한다.

```text
[도메인 담당의 현장 실무 경험]
        ↓
현장 Workflow / 용어 / 문제 가설
        │
        │ + 별도 검증
        │
        ├──────────────┐
        ▼              ▼
현직 사용자 조사     기존 솔루션/논문/사례 조사
        │              │
        └──────┬───────┘
               ▼
        검증된 Pain Point
               ↓
             MVP
```

즉:

- 도메인 담당의 실무 경험 = **강력한 출발점**
- 최신 스마트건설/BIM/3D 관제 실태 = **별도 검증 대상**

---

# 16. 프로젝트 관련 핵심 질문

최초 질문 후보에서 진짜 중요했던 것만 추렸다.

## 16.1 데이터·현장성

- 실제 CCTV/현장 영상 데이터를 확보했는가?
- 공개 데이터셋 중심인가?
- 별도 촬영인가?
- 실제 현장의 불편함을 검증할 경로가 있는가?
- 안전관리자 / 시공담당 / BIM담당 인터뷰가 가능한가?

## 16.2 AI 범위

- Detection / Fine-tuning까지만 하는가?
- Tracking까지 포함하는가?
- Segmentation은 실제 어떤 문제를 해결하기 위해 쓰는가?

## 16.3 Digital Twin

- Pixel → World / Floorplan 변환까지 하는가?
- 단순 bbox 표시인가?
- 실제 객체 위치·상태를 실시간 동기화하는가?

## 16.4 실시간 처리

- RTSP/CCTV 실시간인가?
- 업로드 영상 분석인가?
- AI → Backend → FE 상태 전달까지 실시간인가?

## 16.5 팀 구성

- AI 몇 명인가?
- Model/Data와 Pipeline/Spatial을 분리할 수 있는가?
- Infra/배포 담당은 별도인가?

가장 핵심 6개:

1. 실제 현장 문제를 사용자에게 검증할 수 있는가?
2. Tracking까지 하는가?
3. Pixel → World까지 하는가?
4. Digital Twin을 실시간 상태 동기화 수준으로 구현하는가?
5. AI 인원과 역할을 어떻게 나누는가?
6. 실제 영상 데이터와 실시간 입력 환경이 있는가?

---

# 17. 팀장의 실제 프로젝트 구상 답변

팀장은 다음 3개의 공백을 문제로 제시했다.

## 17.1 무겁고 깨지기 쉬운 3D BIM

- IFC 변환 과정의 스키마 문제
- 좌표 정합 문제
- 현장에서 3D Twin을 제대로 쓰지 못함
- 실제로는 CCTV 분할 화면 의존
- 해결:
  - 현장 도면 2D/2.5D
  - 경량 공간 모델
  - 비전 카메라 좌표 역투영

## 17.2 단순 객체 탐지의 오탐·맥락 부재

예:

- 흙더미 그림자를 crack으로 오인
- 정상 신호수 지휘를 중장비 접근 위험으로 오인
- 수백 번의 false alarm
- 현장에서 AI를 꺼버림

해결 방향:

- YOLO 단순 탐지 이상
- 작업 맥락 이해
- 추론
- VLM
- 상태 처리 파이프라인

## 17.3 감지와 조치 사이 단절

- 알람 후 수기 보고서
- 영상 되돌려보기
- 비효율
- 해결:
  - 감지 즉시 도면에 위치 기록
  - Incident
  - 사후 조치 보고서
  - 안전 일지 자동 생성

---

# 18. 이 답변에서 확인된 프로젝트 정체성

단순:

> 건설현장 YOLO 프로젝트

가 아니라:

> **Perception → Spatial Context → Risk Decision → Action Automation**

프로젝트로 봐야 한다.

세 문제는 다음처럼 연결된다.

```text
WHERE?
공간 정합
   ↓
WHAT DOES IT MEAN?
맥락 추론
   ↓
WHAT DO WE DO?
조치 자동화
```

---

# 19. VLM 적용 방향

VLM을 모든 프레임에 넣는 것은 비효율적이다.

비추천:

```text
Camera
→ VLM
→ 위험 판단
```

추천:

```text
YOLO
 ↓
Tracking
 ↓
Geometry / State Filter
 ↓
위험 후보만 선별
 ↓
VLM
 ↓
확인 / 설명 / 보고서
```

즉 VLM은:

> **1차 감지기보다 2차 맥락 판별기**

로 사용하는 것이 현실적이다.

---

# 20. 최신 스마트건설 / Digital Twin 실무 조사

조사 결과 핵심 결론:

> **"3D/BIM이 무거워서 문제"는 부분적으로 맞지만, 진짜 현장 문제는 데이터 분절, 상호운용성, 좌표 정합, 초기 설정, 교육 비용, 모바일 성능, 오탐, 조치 Workflow 단절에 더 가깝다.**

따라서 프로젝트의 더 강한 포지션은:

> **3D를 없애는 것 자체가 아니라, 안전관리자가 실제 필요한 공간 정보만 2D/2.5D에 남기고 Vision → 공간 정합 → 맥락 판단 → 조치/보고까지 연결하는 것**

이다.

---

# 21. 현장 실무자들이 사용하는 주요 툴

## Bluebeam Revu

주 사용자:

- PM
- Estimator
- Engineer

용도:

- PDF 도면 검토
- 마크업
- 비교

장점:

- 빠른 도면 검토
- 단순함
- 현장 친화적

한계:

- 큰 파일 / 낮은 사양에서 성능 저하
- 기능 학습량

교훈:

> 현장 사용자는 정보량보다 필요한 정보를 빠르게 찾는 것을 중요하게 여긴다.

---

## Fieldwire

주 사용자:

- PM
- Project Engineer
- Superintendent

용도:

- 도면
- Task
- Punch
- 사진
- 이슈 핀

장점:

- 모바일 접근
- 도면 위 Task
- 짧은 학습곡선

한계:

- 커스터마이징
- 보고서 제한
- 대형 도면
- 동기화 성능

교훈:

```text
복잡한 3D
보다

도면
+ 정확한 위치
+ Issue
+ 사진
+ 담당자
+ 상태
```

만으로도 현장 가치가 높다.

---

## Procore

용도:

- RFI
- Drawing
- Daily Log
- 사진
- 비용
- 통합 PM

장점:

- 정보 일원화

한계:

- 가격
- onboarding
- 기능 과다
- 작은 현장에는 무거움

교훈:

> Enterprise Platform 전체 기능이 현장 안전관리 업무와 항상 맞지는 않는다.

---

## Revizto

주 사용자:

- BIM/VDC Coordinator

용도:

- Clash
- Issue
- 3D 협업

장점:

- 2D/3D 이슈 중앙화

한계:

- 대형 모델 로딩
- 고사양 요구
- 초기 구성

---

## Dalux

주 사용자:

- BIM Manager
- 현장

용도:

- 3D Viewer
- Task
- Checklist

장점:

- 모바일 BIM
- 비교적 직관적

한계:

- 초기 설정
- 기능별 제약

교훈:

> "3D는 무조건 현장에서 못 쓴다"는 주장은 과도하다.

---

## Navisworks

용도:

- 통합 모델
- Clash
- BIM Coordination

장점:

- 다양한 설계 모델 통합

한계:

- 대형 NWD/NWC 성능
- 모델 관리 복잡성

---

## OpenSpace

용도:

- 360° 현장 기록
- Floorplan mapping

장점:

- 원격 확인
- 빠른 기록

한계:

- 360 이미지와 평면도 오정합
- 외부 영역 관리

중요한 교훈:

> Camera / Reality 데이터를 평면도에 정확히 정합하는 문제가 실제 상용 제품에서도 품질을 좌우한다.

---

## CupixWorks

용도:

- 360° + BIM / Spatial Twin

장점:

- 현장과 모델 비교
- 원격 확인

한계:

- 촬영 자체가 노동
- 정합 관리
- 층/도면 관리

교훈:

> 3D 모델 생성 비용을 줄여도 현실 데이터 획득·정합 비용은 남는다.

---

# 22. 국내 대형 건설사 흐름

## 현대건설

공개 사례에서 다음을 활용:

- Digital Twin
- 이동식 AI CCTV
- 중장비 충돌/협착 방지
- 작업자 위치관제
- 가스감지
- AIoT 센서

즉 대기업은 이미:

```text
Digital Twin
+ CCTV
+ AI
+ 위치 추적
+ 센서
+ 통합관제
```

까지 진행 중이다.

따라서 프로젝트 차별화는:

> "이 기술을 처음 만든다"

가 될 수 없다.

---

## 삼성물산

공개 방향:

- 드론 점검
- AI 기반 중장비 위험 알림
- S-TBM
- 작업 전 위험 대책 공유
- 안전 workflow 디지털화

최근 흐름은 alarm fatigue를 줄이기 위한 단계적 알림 등 **맥락 기반 안전 판단** 쪽으로 이동한다.

---

## 대우건설

Q-BOX 등에서:

- 모바일/태블릿 입력
- 전자결재
- 시험성적서 자동 매핑
- 정부 시스템 자동 등록

등으로 문서/승인/보고 Workflow 자동화.

중요한 교훈:

> 감지보다 감지 후 행정·조치 업무 자동화가 실제 가치가 클 수 있다.

---

# 23. 기존 가설에 대한 재평가

| 가설 | 평가 |
|---|---|
| 대형 3D/BIM은 현장에서 너무 무겁다 | 부분적으로 맞음 |
| IFC/좌표 정합 문제가 있다 | 명확히 존재 |
| 단순 AI 탐지는 오탐 때문에 현장성이 낮다 | 강하게 타당 |
| 감지 후 보고/조치가 단절된다 | 매우 타당 |
| 2D/2.5D이면 차별화된다 | 그 자체로는 부족 |

---

# 24. 프로젝트의 더 강한 정의

약한 정의:

> 무거운 BIM을 대신하는 가벼운 2D/2.5D 건설 Digital Twin.

더 강한 정의:

> **기존 2D 현장도면을 기준 공간으로 사용해 CCTV Vision으로 관측한 작업자·장비·위험요소를 실시간 정합하고, 시간·공간·작업 맥락으로 오탐을 걸러낸 뒤 Incident와 안전보고까지 자동 연결하는 경량 현장 안전 Spatial Twin.**

참고할 제품 철학:

```text
Fieldwire
도면 중심 현장 UX
       +

OpenSpace / Cupix
Reality ↔ Spatial Mapping
       +

AI CCTV
위험 객체 감지
       +

Context AI
오탐 감소
       +

Q-BOX식 Workflow
보고/행정 자동화
       =
프로젝트
```

---

# 25. MVP 방향

조사 결과를 반영한 MVP:

1. PDF/DWG 도면 업로드
2. 카메라 위치 / 촬영 영역 등록
3. 작업자 / 중장비 / PPE 탐지
4. Tracking
5. Camera → Floorplan Calibration
6. 도면 위 실시간 객체 위치 표시
7. 위험구역 진입 또는 중장비 접근
8. 일정 시간 지속 시 Incident
9. frame/clip + 시간 + 도면 위치 + 판단 근거 저장
10. 안전일지 / 보고서 초안 자동 생성
11. VLM은 후보 사건의 추가 맥락 판단에 선택적으로 사용

---

# 26. 균열 vs 안전모/중장비

둘은 성격이 다르다.

## 안전모 / 중장비

```text
Video
→ Person / PPE / Equipment Detection
→ Tracking
→ Spatial Location
→ Context
```

실시간 동적 관제 문제.

## 균열

```text
Inspection Image
→ Crack Detection / Segmentation
→ 위치
→ 길이 / 면적
→ 과거와 비교
```

정적/주기적 Inspection 문제.

균열 진행까지 보면:

```text
t1 crack
   ↓
t2 crack
   ↓
변화량 비교
```

따라서 MVP에서는:

- 안전 관제 = 핵심
- 균열 = 후속 Inspection 기능

으로 분리하는 것이 자연스럽다.

---

# 27. 현재 실제 팀 구성 구상

현재 팀장이 생각해온 분배:

```text
FE 1

BE 3
- 완전 Backend 2
- AI 파트와 붙는 Backend 1

AI 2
```

추가 정보:

- 팀장은 **AI 파트와 붙어있는 BE** 희망
- AI 모델 학습 담당 1명 이미 모집
- 합류 인원 1명은 AI Pipeline / Spatial Vision

가장 자연스러운 구성:

```text
FE 1
└─ 2D/2.5D Spatial Twin + 관제 UI

BE 3
├─ BE-1 : Domain / API / DB
├─ BE-2 : Backend + Infra / Platform
└─ BE-3 : AI-linked Backend ← 팀장

AI 2
├─ AI-1 : Model / Data / Fine-tuning ← 이미 모집
└─ AI-2 : Vision Pipeline / Spatial Vision ← 합류 확정
```

---

# 28. AI-linked Backend와 Vision Pipeline 역할 경계

모델 담당:

```text
Dataset
→ Fine-tuning
→ Detection / Segmentation
```

Vision Pipeline / Spatial Vision 담당:

```text
Video
→ Inference Runtime
→ Tracking
→ Calibration
→ Pixel → World
→ Spatial Observation
```

팀장 / AI-linked BE:

```text
Observation
→ Context
→ State
→ Risk
→ VLM
→ Incident
```

일반 BE:

```text
Incident
→ DB
→ History
→ API
→ Report
```

FE:

```text
Twin State
→ 2D/2.5D
→ Alert / Report
```

---

# 29. 팀 구성에서 Infra 배치

현재 프로젝트 규모를 보면 별도 Infra 전담은 필수는 아닐 가능성이 높다.

가장 자연스러운 구조:

```text
BE-2
Domain / DB / API
+
공통 Infra / Deploy
```

Vision Pipeline 담당은 자기 Vision Runtime 쪽:

- Docker
- GPU
- CUDA
- model artifact
- health
- AI metrics

을 소유할 수 있다.

공통 Infra:

- Spring runtime
- DB / Redis
- FE deploy
- nginx
- TLS
- common CI/CD

는 BE/Platform 쪽과 협업.

---

# 30. 팀장과의 첫 구체 회의에서 해야 할 일

회의 목표는 세 가지다.

1. 진짜 Pain Point 확인
2. MVP 확정
3. 역할 / 학습 / 팀빌딩 고정

## 반드시 확인할 것

- 가장 불편했던 실제 현장 상황 2~3개
- 실제 사용자는 누구인가?
- 현재는 어떻게 처리하는가?
- 시간/실수/피로가 가장 큰 지점은 어디인가?
- 현직 사용자 인터뷰 경로가 있는가?
- "이 기능 하나만 제대로 되면 가치 있다"는 기능은 무엇인가?

---

# 31. 첫 회의 산출물

회의 종료 시 최소:

1. 핵심 사용자 1명
2. 핵심 Pain Point 1~2개
3. MVP Demo Scenario 1개
4. 입력 데이터 형태
5. AI/Vision 범위
6. Digital Twin 범위
7. Vision Pipeline 담당 역할 경계
8. 추가 모집 직군
9. 기술 검증 항목
10. 다음 회의 전 PoC / 조사 과제

---

# 32. ASC 모니터링 / 작업 큐 철학 적용 가능성

가능하며 매우 잘 맞는다.

핵심 철학:

- Observation과 Execution 분리
- State를 정본으로 둠
- Queue는 정본이 아님
- Detection ≠ Incident ≠ Notification
- 느린 작업은 Work Plane으로 분리
- Backpressure / Queue saturation 관찰
- 자원에 따라 동시성 조절

---

# 33. Realtime Plane vs Work Plane

```text
             REALTIME PLANE
────────────────────────────────

Camera
 ↓
Detection
 ↓
Tracking
 ↓
Spatial State
 ↓
Backend Twin State
 ↓
FE

        │
        │ candidate
        ▼

             WORK PLANE
────────────────────────────────

Risk Queue
 ↓
Context / VLM
 ↓
Incident Queue
 ↓
Evidence 정리
 ↓
Report Queue
 ↓
보고서 생성
```

핵심:

> **보고서 생성이나 VLM이 느려도 실시간 Tracking은 계속 살아 있어야 한다.**

---

# 34. Detection ≠ Incident ≠ Notification

한 프레임에서 위험해 보인다고 바로 HIGH 알림을 띄우지 않는다.

```text
Observation
→ Candidate
→ 일정 시간 / 공간 조건 확인
→ Context 판단
→ Confirmed Incident
→ Notification
```

예:

```text
NORMAL
 ↓
POTENTIAL_RISK
 ↓
CONFIRMED_RISK
 ↓
RESOLVED
```

---

# 35. VLM Work Queue

VLM을 매 프레임 실행하지 않고 후보만 넣는다.

```text
30 FPS Camera
   ↓
YOLO / Tracking
   ↓
Geometry / State Filter
   ↓
위험 후보만 선별
   ↓
VLM Queue
   ↓
Context 판단
```

예:

```json
{
  "jobId": "risk-812",
  "type": "CONTEXT_VERIFY",
  "priority": "HIGH",
  "workerId": 17,
  "equipmentId": 4,
  "spatialState": {
    "distance": 2.7,
    "zone": "A"
  }
}
```

Queue는 정본이 아니다.

```text
Queue
= "이 상태를 추가 조사해야 한다"

Twin State
= 실제 시스템의 정본
```

---

# 36. 동적 동시성 적용

관측 대상:

- Queue depth
- oldest job age
- GPU utilization
- VRAM
- inference latency
- Camera frame delay

예:

```text
평상시
VLM concurrency = 1

Queue 증가 + GPU 여유
→ 2
→ 3

GPU saturation / Vision FPS 저하
→ 2
→ 1
```

우선순위:

```text
Camera / Tracking
        >
Risk candidate processing
        >
VLM enrichment
        >
Report generation
```

보고서 때문에 실시간 Tracking FPS가 떨어지면 안 된다.

---

# 37. 프로젝트 Monitoring 항목

| 계층 | 핵심 관측 |
|---|---|
| Camera | 연결 여부, 입력 FPS, frame lag |
| Inference | FPS, latency, GPU/VRAM |
| Tracking | active tracks, ID switch, lost |
| Spatial | calibration 상태, 좌표 이상치 |
| Queue | depth, oldest job age, retry |
| AI→BE | latency, 실패율 |
| Context/VLM | 처리시간, backlog |
| 전체 | Camera → Twin end-to-end latency |

중요:

> Queue가 쌓이는 것 자체가 장애는 아니다.

예:

- Report queue 지연 → 관제는 정상일 수 있음
- Camera lag 증가 → 실시간 품질 문제

즉 단일 UP/DOWN이 아니라 **파이프라인별 건강 상태**가 필요하다.

---

# 38. 현재 Vision Pipeline 담당의 최종 직군 표현

가장 정확한 표현:

- **직군:** AI / Computer Vision
- **주 역할:** Vision Pipeline / Spatial AI
- **설명:** 모델 추론 결과를 Tracking·좌표 변환·상태 처리까지 거쳐 Backend/Digital Twin에 연결하는 실시간 파이프라인 담당

팀 역할표용:

> **AI - Vision Pipeline**

조금 전문적으로:

> **Vision Systems / Spatial AI Engineer**

가장 자연스러운 한 줄:

> **AI/Vision 파트 — 실시간 Vision Pipeline, Tracking 및 Spatial Mapping 담당**

---

# 39. 현재 프로젝트의 가장 강한 제품 정의

최종적으로 가장 강하게 정리되는 한 문장:

> **기존 2D 현장도면을 기준 공간으로 사용해 CCTV Vision으로 관측한 작업자·장비·위험요소를 실시간 정합하고, 시간·공간·작업 맥락으로 오탐을 걸러낸 뒤 Incident와 안전보고까지 자동 연결하는 경량 현장 안전 Spatial Twin.**

---

# 40. 현재 Vision Pipeline 담당의 역할 한 문장

> **CV 모델 자체를 만드는 것보다, 모델이 인식한 현실 영상을 Tracking·공간 좌표화·실시간 파이프라인을 거쳐 Digital Twin이 사용할 수 있는 상태로 만드는 Vision Systems / Spatial AI를 주 영역으로 가져간다.**

Infra까지 포함하면:

> **Vision System이 GPU 서버에서 실제로 안정적으로 계속 돌도록 AI Runtime까지 책임진다.**

---

# 41. 현재까지 확인된 리스크

1. 팀 도메인 담당의 현장 경험은 강하지만 최신 3D/BIM 관제 제품 실사용 경험은 없음
2. "3D가 무거우니 2D/2.5D"만으로는 차별화 부족
3. 균열과 PPE/중장비는 AI 문제 성격이 다름
4. VLM을 실시간 핵심 경로에 넣으면 latency/비용 리스크
5. FE 1명이 모든 Spatial/UI를 가져가면 범위 과대 가능
6. BE 3명은 역할 분리가 명확해야 함
7. AI 2명은 Model vs Vision Systems 경계를 명확히 해야 함
8. 실제 현장 데이터/인터뷰 경로 확보 필요
9. MVP를 너무 넓게 잡으면 기술 과시형 프로젝트가 될 위험
10. 정확한 Spatial Mapping 품질이 실제 제품 가치에 매우 중요

---

# 42. 현재까지의 가장 합리적인 팀 구조

```text
[Team Lead]
Domain / Product / AI-linked Backend
          │
          ├─────────────────────┐
          │                     │
          ▼                     ▼
      [AI-1]                [AI-2]
   Model / Data         Vision Systems
 Fine-tuning / Seg       Spatial Vision
          │                     │
          └────────┬────────────┘
                   ▼
            Spatial Observation
                   ▼
        [AI-linked Backend]
     Context / State / Risk / VLM
                   ▼
            [Backend Core]
       DB / History / API / Report
                   ▼
              [Frontend]
        2D/2.5D Twin / Alert / UX

     ────── Infra / Runtime ──────
  AI Runtime: AI-2 중심
  Common Infra: BE/Platform 중심
```

---

# 43. 전체 프로젝트의 철학

이 프로젝트가 강해지기 위한 핵심 철학은 다음과 같다.

```text
무거운 모든 것을 보여주는 시스템
        ↓
필요한 것만 정확하게 보여주는 시스템

단순 감지
        ↓
맥락 있는 판단

알람
        ↓
조치

시각화
        ↓
업무 Workflow

AI 모델
        ↓
실제로 계속 돌아가는 시스템
```

---

# 44. 다음 단계

가장 가까운 다음 액션은 다음과 같다.

1. 팀장과 대면/회의
2. 실제 Pain Point 구체화
3. 현직 사용자 인터뷰 가능성 확인
4. MVP 시나리오 하나 고정
5. Camera / Data source 고정
6. Model / Vision / Backend 경계 확정
7. FE 범위 확정
8. Infra 최소 구조 확정
9. Vision Pipeline 담당 Tracking / OpenCV / Homography 선행학습
10. 작은 Camera → Tracking → Floorplan PoC 시작

---

# 45. 조사에서 언급된 주요 제품/자료

아래는 세션 중 실무 조사에서 다룬 대표 제품 및 사례다.

- Bluebeam Revu
- Fieldwire
- Procore
- Autodesk Construction Cloud / Forma
- Revizto
- Dalux
- Navisworks
- OpenSpace
- CupixWorks
- 현대건설 스마트건설 / Digital Twin / AI CCTV
- 삼성물산 AI 안전 / S-TBM
- 대우건설 Q-BOX
- 건설 안전 Digital Twin 연구
- Context-aware Vision / VLM 기반 안전 판단 연구
- IFC / Georeferencing / Interoperability 연구

---

# 46. 최종 요약

현재 프로젝트는 다음처럼 보는 것이 가장 정확하다.

```text
건설 현장
   ↓
CCTV / 이미지
   ↓
AI Model
   ↓
Vision Pipeline
   ↓
Tracking
   ↓
Camera → World
   ↓
Spatial Twin
   ↓
Context / Risk
   ↓
Incident
   ↓
Report / Action
```

Vision Pipeline 담당의 역할:

```text
AI Model 담당
      ↓
★ AI-2: Vision Pipeline / Spatial Vision / AI Runtime ★
      ↓
AI-linked Backend
```

즉 이 역할이 주 무대로 가져갈 전문성은:

> **Computer Vision + Spatial Computing + Realtime Systems**

이다.
