# SENTIA 프로젝트 개요

> **SENTIA — SENTient Intelligence & Action**
>
> *Sense. Understand. Act.*
>
> 건설 현장의 Vision AI 감지 데이터를 공간·맥락·작업 상태와 결합하여 실제 위험 이벤트로 판단하고, 안전 관리자와 작업자에게 즉시 전달해 조치·검증·기록·보고까지 자동 연결하는 경량 스마트 안전 운영 플랫폼

SENTIA는 단순 객체 탐지나 CCTV 대시보드가 아니다.
현장에서 관측한 현실 정보와 작업계획·위험성평가 같은 업무 맥락을 연결하고, AI가 만든 후보를 사람이 검토한 뒤 실제 조치와 기록까지 이어 주는 운영 시스템이다.

핵심은 다음 세 질문을 하나의 흐름으로 연결하는 것이다.

~~~text
무슨 일이 보이는가?
        ↓
실제로 위험한 상황인가?
        ↓
어디에서 발생했고 무엇을 해야 하는가?
        ↓
누가 조치했고 결과를 어떻게 남길 것인가?
~~~

## 배경

건설 현장에는 AI CCTV, IoT 센서, 작업자 위치 관제, BIM·Digital Twin 같은 스마트 안전 기술이 빠르게 도입되고 있다.
그러나 실제 운영에서는 기술을 도입한 것과 안전 업무가 매끄럽게 돌아가는 것이 같은 문제가 아니다.

현재 논의에서 반복해서 확인된 핵심 문제는 다음 세 가지다.

### 1. 오탐과 맥락 부족

현재 Vision AI는 사람, 장비, 보호구, 균열처럼 **보이는 것**을 인지하는 데 강하지만, 그것이 실제 현장 맥락에서 위험한 상황인지 판단하는 데 한계가 있다.

예를 들면 다음과 같다.

- 정상적인 신호수 유도를 중장비 접근 위험으로 판단
- 짧은 보호구 미착용 상태를 반복 경보
- 타설 흔적, 그림자, 조명 변화 등을 구조 결함으로 오인
- 새로운 현장·텍스처·카메라 조건에서 일반화 성능 저하

이런 오탐과 무분별한 알람이 반복되면 관리자는 경보를 신뢰하지 않게 되고, 결국 관제 기능 자체를 꺼 버리는 양치기 소년 효과가 생길 수 있다.

SENTIA는 한 번의 Detection을 곧바로 현장 알람으로 만들지 않는다.

~~~text
Detection / Anomaly
        ↓
Event Candidate
        ↓
Event Queue
        ↓
공간 · 시간 · 작업 맥락
+ 필요 시 VLM
        ↓
Priority / Risk State
        ↓
관리자 1차 검토
   ↙           ↘
 승인           기각
  ↓              ↓
실제 조치      False Alarm
~~~

즉 **AI가 고른 후보와 사람이 승인한 실제 조치 대상을 구분**한다.
기각된 후보는 False Alarm으로 남겨 운영 품질을 확인할 수 있게 하며, 필요하면 향후 Rule·Threshold·Hard Negative·모델 개선 자료로 사용할 수 있다.
자동 재학습 자체를 전제로 하지는 않는다.

### 2. 공간 정보 단절

현장 관제 데이터는 CCTV 화면, 도면, BIM, 점검 이미지, 센서처럼 서로 다른 공간 표현에 흩어져 있다.
관리자는 여러 CCTV 분할 화면을 보며 위험이 어느 층의 어느 구역에서 발생했는지 직접 해석해야 하는 경우가 많다.

반대로 고도화된 3D/BIM 시스템은 좌표 정합, IFC 변환, 초기 구축, 잦은 현장 변경, 렌더링 비용과 같은 운영 부담을 가질 수 있다.

SENTIA는 화면 자체를 공간의 정본으로 사용하지 않고 **현장 좌표를 공통 기준**으로 둔다.

~~~text
Camera / Image / Sensor
        ↓
관측 위치
        ↓
Site / Floor / Zone
        ↓
Spatial State
        ↓
2D / 2.5D 관제
        ↓
필요 시 BIM / 3DGS / Unreal 확장
~~~

따라서 관리자는 단순히 “CAM-03에서 경보가 발생했다”가 아니라 **어느 현장·층·구역에서 어떤 작업 중 어떤 위험이 발생했는지** 확인할 수 있다.

### 3. 조치·기록 단절

위험을 탐지한 뒤 실제 현장에서는 다시 사람이 영상을 확인하고, 전화·무전·메신저로 상황을 전달하고, 조치 결과를 엑셀·HWP·수기 문서에 옮겨 적는 일이 반복될 수 있다.

문제는 알람 자체보다 다음과 같은 연결부다.

~~~text
감지
→ 확인
→ 현장 전달
→ 조치
→ 조치 확인
→ 증빙
→ 일지 / 보고
→ 위험성평가 피드백
~~~

SENTIA는 승인된 사건을 Action으로 연결하고, 현장 담당자가 이를 수신·확인하며, 조치 결과와 Evidence를 다시 시스템으로 돌려보내는 구조를 지향한다.

또한 Event, Action, ACK, Verification, Evidence, History를 이미 구조화해서 보유하고 있으므로 같은 정보를 사람이 다시 문서에 옮기지 않도록 **AI 보고서 초안 생성**으로 연결한다.

AI가 최종 문서를 법적·운영상 판단 없이 자동 확정하는 것이 아니라 다음 구조를 기본으로 한다.

~~~text
History / Evidence
        ↓
AI Report Draft
        ↓
안전관리자 검토 · 수정
        ↓
안전일지 / 조치보고 /
아차사고 기록 / 위험성평가 피드백
~~~

## 세 Pain Point와 제품 대응

| Pain Point | SENTIA의 직접 대응 |
|---|---|
| 오탐·맥락 부족 | Event Candidate → Event Queue → Context/VLM/Spatial → Priority/Risk State → 관리자 승인/기각 |
| 공간 정보 단절 | Site/Floor/Zone 기반 Spatial State와 2D/2.5D 공간 관제 |
| 조치·기록 단절 | Action Delivery → ACK/Verify → Evidence/History → AI Report Draft |

이 세 축은 별개의 부가기능이 아니라 SENTIA가 하나의 제품으로 성립하기 위한 기본 구조다.
구체적인 사건 종류, 단말, 보고서 형식과 구현 깊이는 04-scope-and-plan.md에서 팀 합의로 정한다.

## 대상 사용자

### Primary — 안전관리자

안전관리자는 SENTIA의 핵심 사용자다.

주요 역할은 다음과 같다.

- 작업계획과 위험성평가 확인
- Event Queue와 공간 관제 확인
- 후보 위험 승인 또는 기각
- 현장 조치 요청
- 조치와 증빙 검증
- 안전일지·보고서 초안 검토

주요 인터페이스는 **도면 기반 관제센터 화면**이며, 즉시성 높은 알림을 위한 스마트워치·모바일 형태도 제품 방향에 포함한다.

### 현장 작업자

작업자는 승인된 조치의 실제 수행 주체다.

제품 방향에서는 스마트워치처럼 작업 중 확인 부담이 낮은 단말을 통해 다음 흐름을 연결한다.

~~~text
조치 수신
→ ACK
→ 현장 대응
→ 완료 / 증빙
~~~

이번 프로젝트에서 실제 스마트워치 Native 앱까지 구현할지, 모바일·웹·모의 Endpoint로 검증할지는 팀의 범위 결정 사항이다.
다만 **관리자 화면에서 Action을 생성하고 끝나는 것이 아니라 현장 전달과 응답까지 이어져야 한다는 제품 방향**은 유지한다.

### Secondary

- 관리감독자·공사 담당자: 작업 지휘와 현장 조치
- 관제 운영자: CCTV·센서·스마트 안전장비 운영
- 프로젝트·본사 담당자: 이력과 보고 결과 활용

## 현장 업무

안전관리 업무는 하루의 운영 흐름으로 연결된다.

| 시점 | 주요 업무 |
|---|---|
| 작업 전 | 작업계획 → 위험성평가 → TBM |
| 작업 중 | 관측 → 위험 판단 → 승인·기각 → 조치 |
| 작업 후 | 조치 검증 → Evidence → 기록·보고 → 위험성평가 피드백 |

SENTIA의 핵심은 CCTV를 계속 보는 일이 아니라 **실제 현장 상태를 선별하고, 사람이 판단하고, 현장에 전달하고, 결과를 기록하는 업무 루프를 줄이는 것**이다.

## 핵심 서비스 흐름

~~~text
[업무 Context]
작업계획 · 위험성평가 · TBM
             │
             ▼
Camera / Sensor / Inspection
             │
             ▼
   Detector / Anomaly
             │
             ▼
      Event Candidate
             │
             ▼
        Event Queue
             │
             ▼
Context / VLM / Spatial State
             │
             ▼
     Priority / Risk State
             │
             ▼
        관리자 검토
        ↙       ↘
     승인        기각
      │           │
      │       False Alarm
      │           │
      ▼           └────→ 운영 품질 / 개선 근거
  Safety Event
      │
      ▼
     Action
      │
      ▼
현장 전달 / 담당자 수신
      │
      ▼
    ACK / 대응
      │
      ▼
     Verify
      │
      ▼
Evidence / History
      │
      ▼
 AI Report Draft
      │
      ▼
 관리자 검토·수정
      │
      ▼
일지 / 보고 / 위험성평가 Feedback
~~~

관리자 승인 전의 후보와 승인 후의 Actionable Event를 구분하는 것이 중요하다.
AI는 검토해야 할 상황을 선별하지만 최종 업무 판단은 사람이 한다.

## 실시간 관제와 판단 작업의 분리

실시간 Spatial State와 느린 맥락 판단·문서화는 같은 경로를 공유하지 않는다.

~~~text
REALTIME
Camera → Detection → Tracking → Spatial State → Realtime → FE

WORK
Event Candidate → Queue → Context/VLM → Review
                                  ↓
                             Action / Report
~~~

VLM이나 보고서 생성이 느려져도 실시간 Tracking과 공간 관제가 멈추지 않아야 한다.

Notion에서 제시된 Hybrid Edge AI 방향도 이 철학과 맞는다.
현장 또는 가까운 실행 환경에서 1차 탐지를 빠르게 수행하고, 무거운 VLM·문서화 작업은 후보에 한해 별도 Work Plane에서 실행할 수 있다.
구체적인 Edge 장비와 배포 형태는 프로젝트 범위에서 결정한다.

## 공간 관제

공간 데이터의 정본은 화면 픽셀이 아니다.

~~~text
Camera Pixel
   ↓
Drawing Space
   ↓
Site Local
   ↓
Floor / Zone
~~~

2D와 2.5D는 안전관리자가 현재 위치와 위험을 빠르게 이해하기 위한 핵심 운영 인터페이스다.

표현할 수 있는 대상은 다음과 같다.

- 작업자와 장비
- Camera와 촬영 영역
- Zone과 위험구역
- Event Candidate와 승인된 Event
- 구조물 결함
- Sensor 상태
- 현장 변화와 Reality Snapshot

실제 제품 범위는 04에서 결정한다.

## AI 보고서

보고 자동화는 별도 장식 기능이 아니라 세 번째 Pain Point를 닫는 업무 흐름의 마지막 단계다.

보고서의 근거는 자유 생성 텍스트가 아니라 시스템이 이미 확보한 구조화 정보다.

- Event와 관리자 승인·기각 기록
- 위치와 작업 Context
- Action과 담당자
- ACK와 Verification
- Evidence
- 시간과 처리 이력
- False Alarm 통계

이 정보를 기반으로 AI가 초안을 만들고, 안전관리자가 검토·수정한 뒤 업무 문서에 활용한다.

구체적인 보고서 종류와 출력 형식은 팀이 정하지만, **History/Evidence → AI Report Draft → Human Review**라는 제품 방향은 유지한다.

## 확장 전략

SENTIA의 목적은 무거운 3D 시스템을 새로 만드는 것이 아니다.
공간·사건·시간 상태를 Viewer와 분리해서 관리하고, 필요에 따라 표현 계층을 확장한다.

~~~text
Spatial / Event Core
      │
      ├── Canvas / SVG      2D
      ├── Three.js          2.5D / Lightweight 3D
      ├── 3DGS              Reality Layer
      ├── BIM / Mesh        구조·의미
      └── Unreal            Immersive / Simulation
~~~

Notion의 3DGS 방향은 실제 시공 현장의 시각적 Reality를 가볍게 연결하는 전략적 확장으로 유지한다.
다만 3DGS·BIM·Unreal을 이번 프로젝트에서 어디까지 구현할지는 Core Product 상태와 일정에 따라 결정한다.

## 현재 상태

| 상태 | 내용 |
|---|---|
| 방향 확정 | 세 Pain Point를 모두 제품 구조에서 추적한다<br>Event Candidate와 관리자 승인·기각을 분리한다<br>공간 데이터는 Site/Floor/Zone 기준으로 연결한다<br>승인된 사건은 현장 조치·검증·Evidence로 이어진다<br>History/Evidence를 AI 보고서 초안으로 연결한다<br>최종 위험 판단과 보고서 확정은 사람이 한다<br>Realtime Plane과 느린 Work Plane을 분리한다 |
| 팀 합의 필요 | Main Demo Story의 구체 사건<br>핵심 관측 대상과 Source<br>현장 Action Endpoint의 구현 형태<br>AI 보고서 종류와 출력 형식<br>VLM 적용 사건과 모델<br>2D/2.5D 구현 깊이 |
| 검증 필요 | 모델 성능과 오탐 수준<br>Tracking·Calibration 품질<br>Event Queue 처리 지연<br>현장 전달 방식의 재현성<br>AI 보고서 초안 품질<br>Testbed·실제 데이터 확보 |
| 조건부 확장 | 3DGS<br>BIM 고도화<br>Unreal<br>Thermal·Telematics·Wearable<br>Multi-camera |

이 방향이 정해지기까지의 조사와 논의 과정은 docs/00-discovery/sessions/에 기록되어 있다.
