# SENTIA — 2D/2.5D Digital Twin Core 및 Unreal Expansion 설계 정본

## 0. 문서 목적

본 문서는 현재 프로젝트에서 논의된 다음 내용을 하나의 설계 기준으로 통합한다.

- 국내 건축·건설 산업에서 Unreal Engine의 실제 위치
- Revit, Navisworks, SYNCHRO, Tekla, Cupix, Autodesk Tandem, Bentley iTwin, Twinmotion, Enscape, D5 등 기존 상용 툴과 Unreal의 차이
- 현재 프로젝트에서 2D/2.5D를 초기 및 핵심 제품으로 두어야 하는 이유
- Canvas/SVG/Three.js/Unreal을 하나의 공간 데이터 모델 위에서 자유롭게 교체·확장하는 방법
- 평면도·도면과 실제 공간 좌표를 연결하는 방법
- Unreal을 본체가 아닌 선택형 Expansion Pack으로 도입하는 구조
- 약 8주라는 제한된 프로젝트 기간을 MVP → 핵심 제품 → 확장팩 → 안정화로 나누는 방법
- Unreal 도입 여부를 판단하는 Go / No-Go 기준
- 프로젝트에서 지켜야 할 최종 설계 원칙

---

# 1. 핵심 결론

현재 프로젝트는 처음부터 Unreal 중심으로 설계하지 않는다.

정식 제품의 핵심은 다음이다.

> **2D 또는 2.5D 기반 건설 현장 상황 관제 시스템**

그 위에 필요하면 다음을 붙인다.

> **Unreal Engine 기반 Immersive / Simulation Expansion Pack**

따라서 프로젝트를

> Three.js 프로젝트에 나중에 Unreal을 붙인다.

라고 정의하지 않는다.

대신 다음과 같이 정의한다.

> **하나의 Spatial Digital Twin Core에 Canvas, Three.js, Unreal 등 여러 Viewer를 연결한다.**

기본 관계는 다음과 같다.

```text
                    ┌─ Canvas / SVG
                    │   2D Monitoring
                    │
[Spatial Core] ─────┼─ Three.js
                    │   2.5D / Web 3D
                    │
                    └─ Unreal
                        Immersive / Simulation
```

이 구조에서는 Unreal이 없어도 제품이 완성되어야 한다.

---

# 2. 국내 건축·건설 산업에서 Unreal의 실제 위치

## 2.1 현재 산업 중심은 Unreal이 아니라 BIM

국내 건축·건설 산업에서 디지털 업무의 중심축은 현재도 다음 계열이다.

- BIM
- CAD
- CDE
- 공정 관리
- Reality Capture
- 현장 안전·품질 관리 플랫폼
- 센서/IoT
- 자체 스마트건설 플랫폼

대표적인 기반 툴은 다음과 같다.

- Autodesk Revit
- Autodesk Navisworks
- Autodesk Construction Cloud / BIM 360
- Graphisoft Archicad
- Trimble Tekla Structures
- Bentley SYNCHRO
- Bentley iTwin
- Autodesk Tandem
- 각 건설사 자체 BIM/CDE/현장관리 시스템

Unreal Engine이 이 영역을 직접 대체하는 것은 아니다.

Unreal은 주로 기존 데이터를 받아 다음을 구현하는 상위 애플리케이션 계층에서 강하다.

- 실시간 3D
- 고품질 시각화
- VR/XR
- 사용자 인터랙션
- 시뮬레이션
- 디지털트윈 Viewer
- 관제용 공간 UI
- 교육/훈련
- 멀티유저 공간
- 산업용 3D 애플리케이션

---

# 3. 건설 생애주기별 Unreal의 적합성

| 단계 | 실제 주력 도구 | Unreal의 위치 |
|---|---|---|
| 개념·건축설계 | Revit, Archicad, Rhino, SketchUp | 보조 |
| 구조·MEP·상세 BIM | Revit, Tekla | 비주력 |
| 도면·물량·속성 | BIM 저작툴 | 부적합 |
| 설계 간섭 검토 | Navisworks, Revizto 계열 | 보조 |
| CDE·문서·이슈관리 | ACC/BIM 360/자체 플랫폼 | 부적합 |
| 4D 공정관리 | SYNCHRO/Navisworks | 커스텀 표현 가능 |
| Reality Capture | Cupix, 스캐너, 드론 | 결과 표현 가능 |
| 건축 시각화 | Twinmotion, D5, Enscape, UE | 강함 |
| 분양·VR/XR | Twinmotion, UE, Unity 등 | 매우 강함 |
| 디지털트윈 UI | Tandem, iTwin, 자체 DT, UE | 강함 |
| IoT 3D 관제 | Web 3D, UE, Unity, Omniverse | 강함 |
| 안전·교육 시뮬레이션 | UE, Unity, 전용 시뮬레이터 | 매우 강함 |
| AI + 3D 시뮬레이션 | UE, Omniverse 등 | 성장 영역 |

결론적으로 Unreal은 건축·건설의 기본 생산 툴이라기보다 **고급 실시간 공간 애플리케이션 엔진**에 가깝다.

---

# 4. 주요 상용 툴과 Unreal의 관계

## 4.1 Revit / Archicad

Revit이나 Archicad의 건축 객체는 단순 Mesh가 아니다.

예를 들어 벽 하나에도 다음 정보가 존재한다.

```text
Wall
├─ 구조
├─ 재료
├─ 치수
├─ 화재성능
├─ 공종
├─ 물량
├─ 층
├─ BIM ID
└─ 다른 설비와의 관계
```

따라서 이들은 다음을 위한 도구다.

- 건축 설계
- 구조/MEP
- 파라메트릭 모델링
- 평면도/단면도
- 일람표
- 물량
- BIM 속성
- 설계 문서

반면 Unreal의 객체는 보다 실행 환경에 가깝다.

```text
BuildingElement
├─ Mesh
├─ Material
├─ Transform
├─ Metadata
├─ State
├─ Interaction
└─ Runtime Logic
```

따라서:

> Revit = 건물 정보를 작성하는 시스템  
> Unreal = 작성된 공간을 실행 가능한 실시간 애플리케이션으로 만드는 시스템

이다.

둘은 경쟁관계라기보다 연결 관계다.

```text
Revit / BIM
    ↓
Datasmith 등
    ↓
Unreal
```

---

# 5. Twinmotion의 위치

건축 업계에서는 Unreal 자체보다 Twinmotion이 직접적으로 사용하기 쉽다.

구조를 단순화하면:

```text
Revit
  ↓
Twinmotion
  ↓
렌더 / 프레젠테이션 / VR

필요 시
  ↓
Unreal Engine
  ↓
커스텀 로직 / IoT / AI / 시뮬레이션
```

따라서 단순 시각화만 필요하다면 Unreal을 직접 사용하는 것이 과할 수 있다.

---

# 6. Navisworks와 Unreal

Navisworks는 실제 시공 업무에 훨씬 직접적으로 붙어 있다.

주요 기능:

- 여러 BIM 모델 통합
- Clash Detection
- 공정 연계
- TimeLiner
- 4D 시뮬레이션
- 수량 및 프로젝트 데이터 조회

Unreal에서도 비슷한 화면은 만들 수 있으나 다음 기능을 직접 만들어야 한다.

```text
BIM parser
Schedule mapping
Issue DB
Object mapping
Revision management
Permission
Collaboration
UI
Backend
```

따라서:

> 기존 시공 업무 → Navisworks  
> 새로운 커스텀 공간 서비스 → Unreal 가능

으로 구분한다.

---

# 7. Bentley SYNCHRO 4D와 Unreal

SYNCHRO는 시공 일정과 3D 객체를 연결하는 것이 핵심이다.

예:

```text
2026-10-01
기초 공사

2026-10-15
1층 골조

2026-11-01
2층 골조
```

이 일정을 실제 모델 객체에 연결한다.

Unreal에서도 훨씬 화려하게 표현할 수 있지만 근본 차이는 다음과 같다.

> SYNCHRO = 공정관리 제품  
> Unreal = 공정관리 제품을 새로 만들 수도 있는 엔진

이다.

---

# 8. Tekla Structures와 Unreal

Tekla는 철골·콘크리트·철근 등 실제 제작 가능한 수준의 BIM에 강하다.

따라서 Unreal과 직접 경쟁하지 않는다.

가능한 관계는 다음과 같다.

```text
Tekla
↓
상세 구조 BIM
↓
Unreal
↓
시공 시각화
안전교육
작업자 동선
VR
```

---

# 9. Cupix와 프로젝트의 관계

Cupix 계열의 접근법은 현재 프로젝트를 이해하는 데 중요하다.

기본 구조는 다음과 같다.

```text
360° Camera
↓
Reality Capture / AI
↓
공간 복원
↓
BIM 정합
↓
계획 ↔ 실제 비교
↓
공정 / 이슈 / 변화
```

Unreal은 이 Reality Capture 자체를 담당하는 엔진으로 보기보다는 결과를 소비하는 쪽이 적합하다.

```text
분석 결과
↓
Unreal
↓
3D Digital Twin Viewer
↓
이상 객체 강조
시간축
경보
VR
```

---

# 10. Autodesk Tandem / Bentley iTwin과 Unreal

이들은 Unreal과 비교할 때 Digital Twin 쪽에서 더 직접적인 비교 대상이다.

Tandem/iTwin은 이미 다음을 플랫폼 형태로 제공한다.

- BIM 연결
- 객체 데이터
- IoT
- 센서
- 시계열
- 설비 관계
- 데이터 관리
- 클라우드
- API

반면 Unreal은 이를 자동으로 제공하지 않는다.

대신:

- 렌더링
- 자유로운 UI
- 인터랙션
- 물리
- VR/XR
- 커스텀 애플리케이션

에서 강하다.

정리하면:

> Tandem / iTwin = 완성형 Digital Twin Platform  
> Unreal = Digital Twin Application을 만들 수 있는 실시간 엔진

이다.

---

# 11. Enscape / D5 / Lumion과 Unreal

단순히 건물을 예쁘게 보여주는 것이 목적이라면 Unreal이 반드시 최적은 아니다.

### 적합한 선택 예시

| 목적 | 적합한 도구 |
|---|---|
| Revit 작업 중 즉시 렌더링 | Enscape |
| 빠른 고품질 렌더 | D5 / Enscape |
| 고객 프레젠테이션 | Twinmotion / D5 |
| 영상·렌더 | D5 / Lumion / Twinmotion |
| 간단 VR | Twinmotion / Enscape |
| 복잡한 VR 앱 | Unreal |
| 센서 연동 | Unreal / DT Platform |
| 사용자 인터랙션 | Unreal |
| AI 시뮬레이션 | Unreal / Omniverse |
| 독립 제품 개발 | Unreal / Web 3D |

따라서 Unreal은 단순 Render Tool로 쓰기보다는 **Software Engineering이 개입하는 순간** 가치가 커진다.

---

# 12. Unreal의 핵심 장점

Unreal의 장점은 개발자가 개입할 수 있다는 점이다.

다음 요소들을 하나의 공간 애플리케이션으로 묶을 수 있다.

```text
BIM
+
IoT
+
AI
+
REST
+
WebSocket
+
Database
+
Computer Vision
+
LiDAR
+
GIS
+
Time-series
+
Multi-user
+
VR / XR
```

따라서 특히 다음 영역에서 의미가 커진다.

> AEC × Software Engineering

---

# 13. Unreal의 약점

Unreal은 건설 데이터 플랫폼이 아니다.

예를 들어 하나의 벽을 클릭해서 다음 정보를 보려면:

```text
벽 클릭
↓
BIM GUID
↓
공종
↓
계획 공정
↓
현재 공정률
↓
현장 사진
↓
센서
↓
이상 이력
↓
담당자
```

이 정보를 제공하는 Backend/Data Model은 별도로 구현해야 한다.

따라서 현실적인 전체 구조는:

```text
                         ┌─ BIM / IFC
                         │
Reality Capture ─────────┼─ Point Cloud
                         │
IoT / Sensor ────────────┼─ Time Series
                         │
Schedule / ERP ──────────┼─ Project Data
                         │
AI ──────────────────────┼─ Detection / Prediction
                         ↓
                 Digital Twin Backend
                         ↓
             ┌───────────┴───────────┐
             ↓                       ↓
          Web UI                 Unreal Client
         관제/보고서             3D/XR/Simulation
```

이 구조가 적합하다.

---

# 14. 현재 프로젝트의 기본 제품 철학

현재 프로젝트에서는 2D 또는 2.5D가 초기 컨셉이며, 이것 자체를 축소판이 아니라 정식 제품으로 취급한다.

그 이유는 실제 건설 현장 관리에서 관리자가 가장 먼저 원하는 정보가 다음과 같기 때문이다.

```text
어느 층?
어느 구역?
무슨 이상?
언제 발생?
누가 있음?
얼마나 심각?
```

이 질문은 반드시 3D가 아니어도 훌륭하게 해결할 수 있다.

오히려 평면도가 빠를 수도 있다.

예:

```text
┌──────────── 3F ──────────────┐
│                              │
│   🟢 W12       ⚠ W18         │
│                              │
│       🔴 Crack #C102          │
│                              │
│                 🚧 Crane     │
│                              │
└──────────────────────────────┘
```

여기에 다음 기능을 추가한다.

- Zoom
- Pan
- Zone
- Heatmap
- Worker
- Equipment
- Crack
- CCTV
- Timeline
- Alert

이것만으로도 실제 관제 시스템의 핵심 경험을 구성할 수 있다.

---

# 15. Viewer가 아닌 Spatial Model이 정본이어야 한다

가장 중요한 아키텍처 원칙이다.

잘못된 데이터:

```ts
{
  x: 428,
  y: 163,
  canvasColor: "red"
}
```

이 값은 Canvas 화면에 종속된다.

대신 다음처럼 저장한다.

```ts
{
  id: "worker-102",
  type: "worker",

  spatial: {
    floorId: "F03",
    x: 12.4,
    y: 7.8,
    z: 0,
    coordinateSystem: "SITE_LOCAL"
  },

  state: {
    status: "WARNING",
    helmet: false
  },

  metadata: {
    zoneId: "zone-A",
    detectedAt: "..."
  }
}
```

그러면 렌더러별로 다르게 표현할 수 있다.

```text
Canvas
→ 평면도 (x, y)

Three.js
→ World (x, y, z)

Unreal
→ Unreal Transform
```

Canvas, Three.js, Unreal이 서로 데이터를 변환하지 않는다.

모두 같은 Spatial Core를 읽는다.

---

# 16. Progressive Enhancement 구조

프로젝트의 시각화는 다음 단계로 자연스럽게 확장한다.

## Level 0 — 2D

```text
Floor Plan
+
Marker
+
Zone
+
Alert
```

Canvas / SVG 기반.

---

## Level 1 — 2.5D

```text
Floor Plan
+
높이값
+
층 분리
+
간단 Geometry
+
Object Overlay
```

예:

```text
        ┌─────────┐
        │   4F    │
        └─────────┘

     ┌──────────────┐
     │      3F      │
     └──────────────┘

   ┌─────────────────┐
   │       2F        │
   └─────────────────┘

┌──────────────────────┐
│          1F          │
└──────────────────────┘
```

Three.js의 Orthographic Camera 또는 exploded floor view 등을 사용할 수 있다.

---

## Level 2 — Lightweight 3D

Three.js에 다음을 추가한다.

- glTF
- 간단 BIM geometry
- 장비
- 작업자
- 위험 영역
- 센서
- 이벤트

---

## Level 3 — Unreal Expansion

다음 요구가 있을 때 Unreal을 추가한다.

- 고품질 현장 재현
- 자유 보행
- VR
- 안전교육
- 중장비 시뮬레이션
- 작업자 동선 시뮬레이션
- 물리
- 재난/피난
- Photorealistic Twin
- 복잡한 멀티유저
- Immersive Control Room

구조는 다음과 같다.

```text
2D
↓
2.5D
↓
Web 3D
↓
Unreal
```

각 단계는 이전 단계를 버리지 않는다.

---

# 17. 도면 자체도 Spatial Asset으로 취급

도면을 단순 배경 이미지로 처리하지 않는다.

최소 데이터 구조:

```ts
DrawingLayer {
  id
  floorId

  source:
    image | svg | pdf | cad-derived

  width
  height

  transform: {
    originX
    originY
    scale
    rotation
  }

  worldMapping: {
    worldOrigin
    worldScale
    axis
  }
}
```

예를 들어 1920×1080 PNG가 존재한다.

```text
(0,0)
┌──────────────────────────┐
│                          │
│          ●               │
│                          │
└──────────────────────────┘
                  (1920,1080)
```

이 좌표를 현장 좌표에 정합한다.

```text
pixel(0, 0)
↓ transform
site(125.5m, 83.2m)
```

이 구조가 존재하면 같은 도면 데이터를 Three.js나 Unreal에서도 사용할 수 있다.

---

# 18. 좌표계 계층

건설 Digital Twin에서는 화면 좌표를 데이터 정본으로 사용하지 않는다.

최소한 다음 공간을 분리한다.

```text
Screen Space
↓
Drawing Space
↓
Site Local Space
↓
World Space
```

예를 들어:

```text
Canvas
screenX = 728
screenY = 413
```

는 View State일 뿐이다.

실제 저장 대상은:

```text
SiteLocal
x = 24.42m
y = 18.31m
z = 6.20m
```

이어야 한다.

Renderer별 변환:

```ts
siteToCanvas()
siteToThree()
```

Unreal:

```cpp
SiteToUnreal()
```

이 구조가 플랫폼 독립성을 만든다.

---

# 19. Renderer Adapter

Web Frontend에서는 Renderer를 추상화한다.

예:

```ts
interface SpatialRenderer {
  loadDrawing(layer: DrawingLayer): void;

  addObject(obj: SpatialObject): void;
  updateObject(obj: SpatialObject): void;
  removeObject(id: string): void;

  setCamera(camera: CameraState): void;

  focusObject(id: string): void;
  highlightZone(zoneId: string): void;
}
```

구현체:

```text
CanvasRenderer
ThreeRenderer
```

Unreal도 같은 개념의 API Contract를 따른다.

```text
Spatial Core
    ↓
Renderer Adapter

Canvas
Three.js
Unreal
```

---

# 20. Viewer 간 상태 연속성

2D → 2.5D → Unreal 전환 시 사용자의 현재 맥락을 유지해야 한다.

공통 ViewState 예:

```ts
ViewState {
  floorId: "F03",

  target: {
    x: 31.2,
    y: 18.7,
    z: 6.4
  },

  selectedObjectId: "crack-128",

  timestamp: "2026-09-24T10:30:00Z"
}
```

따라서:

```text
Canvas
↓
3D 보기
↓
Three.js
↓
Immersive 보기
↓
Unreal
```

에서도 동일한:

- Floor
- Object
- Position
- Timestamp

가 열린다.

이것이 Viewer 전환이 아니라 **하나의 Digital Twin을 여러 방식으로 보는 경험**이다.

---

# 21. Unreal은 별도 Client / Expansion으로 분리

프로젝트 구조 예:

```text
platform/
├─ core-api
├─ event-stream
├─ spatial-model
│
├─ frontend-web
│   ├─ canvas-view
│   └─ three-view
│
└─ unreal-client
```

Web 시스템과 Unreal은 같은 Backend를 바라본다.

```text
Backend
├─ REST
├─ SSE
└─ 필요 시 WebSocket / gRPC
```

현재 Web에서 SSE를 사용한다면 Unreal 때문에 초기 통신 구조를 전면 변경할 필요는 없다.

필요할 때 Unreal용 실시간 채널만 별도로 추가하면 된다.

---

# 22. Three.js로 충분한 영역

Unreal을 무조건 쓰지 않는다.

다음은 대부분 Web/Three.js로 충분하다.

- 평면도 관제
- 층별 관제
- CCTV 위치
- 작업자 위치
- 설비 상태
- 이상징후
- 위험 Zone
- 간단 BIM
- 공정 비교
- Heatmap
- 간단한 3D Building
- Object Overlay
- 시간축 기반 상태 확인

---

# 23. Unreal이 의미 있는 영역

다음부터 Unreal의 도입 비용을 지불할 가치가 생긴다.

- 자유로운 현장 보행
- 높은 공간 몰입감
- 실제 현장과 유사한 시각화
- VR 안전교육
- 장비 운전 Simulation
- 작업자 동선 Simulation
- 중장비 충돌 Simulation
- 피난/재난
- Physics
- 대규모 실시간 공간
- 다중 사용자 상황실
- Photorealistic Digital Twin

---

# 24. 프로젝트의 최종 아키텍처 방향

```text
[현장]
CCTV / 360 Camera / Drone / IoT / BIM / CAD / Drawing
                          │
                          ▼
                    [수집 계층]
              Video / Sensor / Spatial
                          │
                          ▼
                     [AI 계층]
         Crack / PPE / Worker / Equipment
         Progress / Anomaly / Tracking
                          │
                          ▼
               [Digital Twin Core]
          Object / Floor / Zone / Event
        Spatial Mapping / State / Timeline
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
          Canvas/SVG   Three.js     Unreal
             2D         2.5D        Expansion
```

핵심은 Backend와 Spatial Core가 Viewer에 종속되지 않는 것이다.

---

# 25. 프로젝트 기간 전제

총 프로젝트 기간은 약 8주다.

다만 다음과 같은 부수 작업이 포함되기 때문에 실제 순수 개발 시간은 8주 전체가 아니다.

- 기획
- 조사
- 데이터 확보
- 팀 협업
- 문서
- 발표
- 운영 이슈
- 환경 구축
- 테스트
- 통합

따라서 실제 개발 가능 기간은 보수적으로 약 6~6.5주 정도로 생각하고 범위를 설정한다.

---

# 26. 전체 프로젝트 진행 원칙

다음 순서로 진행한다.

```text
Core MVP
↓
Core Product
↓
Expansion Pack
↓
Stabilization
```

목표는 특정 기능을 많이 넣는 것이 아니라:

> **가능한 빨리 독립적으로 동작하는 제품을 만든 다음 완성도를 위로 쌓는 것**

이다.

---

# 27. Phase 1 — Core MVP
## Week 1~2

목표:

> **도면을 올리고 → AI/이벤트가 발생하고 → 관리자가 공간 위에서 이상 위치와 상태를 확인한다.**

초기 UI 예:

```text
┌─────────────────────────────────────┐
│ Project / Floor / Date              │
├──────────────┬──────────────────────┤
│              │                      │
│ Event List   │     Floor Plan       │
│              │        🔴            │
│ ⚠ 균열       │                      │
│ ⚠ 안전모     │              🟡      │
│              │                      │
├──────────────┴──────────────────────┤
│ 상세 정보 / 이미지 / 시간 / 상태    │
└─────────────────────────────────────┘
```

최소 기능:

- 프로젝트 선택
- 층 선택
- 도면 선택
- PNG/JPG/PDF 기반 평면도
- 객체 위치
- 이벤트 위치
- AI 탐지 1~2종
- 이벤트 리스트
- 상세 정보
- 시간
- 상태
- 최소한의 실시간 갱신

첫 번째 Vertical Slice:

```text
입력
↓
AI Detection
↓
Event
↓
Backend
↓
Spatial Mapping
↓
2D Viewer
↓
관리자 확인
```

Week 2에는 이 파이프라인이 반드시 완주해야 한다.

---

# 28. MVP에서도 임시로 만들면 안 되는 것

UI는 나중에 갈아도 된다.

하지만 데이터 구조는 처음부터 정본에 가깝게 만든다.

최소 도메인:

```text
Project
└─ Building
   └─ Floor
      ├─ Drawing
      ├─ Zone
      └─ SpatialObject

Event
├─ objectId
├─ floorId
├─ position
├─ type
├─ severity
├─ timestamp
└─ evidence
```

좌표도 최소한:

```text
Drawing Coordinate
↕
Site Local Coordinate
```

를 분리해야 한다.

그래야 이후 2.5D 및 Unreal을 추가할 때 전체를 뜯지 않는다.

---

# 29. Phase 2 — Core Product
## Week 3~5

MVP의 목표가:

> 이벤트가 보인다

라면 Core Product 목표는:

> **현장을 지속적으로 관리할 수 있다**

이다.

---

# 30. Week 3 — Monitoring UX

2D 관제를 제품 수준으로 끌어올린다.

기능:

- Pan
- Zoom
- Event clustering
- Severity
- Zone Overlay
- Floor Switch
- Filter
- Timestamp
- Event History
- 처리 상태
- 미처리 / 확인 / 완료

상태 요약 예:

```text
🔴 Critical  3
🟠 Warning   7
🟡 Observe  12
```

이벤트 상세 예:

```text
균열 #CR-0182

위치
B동 3F / Zone C

발생
11:42:31

AI confidence
94.8%

이전 상태
없음

현재 상태
신규

[현장 이미지]
```

이 시점부터 단순 AI 데모가 아니라 관제 제품처럼 보여야 한다.

---

# 31. Week 4 — Temporal Twin

시간축은 이 프로젝트를 일반적인 AI Detection Dashboard와 구분하는 핵심이다.

일회성:

```text
현재 균열 있음
```

이 아니라:

```text
과거에는 어땠는가
↓
현재 얼마나 변했는가
↓
변화 속도가 비정상인가
```

를 본다.

예:

```text
09/21
────●────────

09/22
──────●──────

09/23
────────●────

09/24
──────────●──
```

균열 예:

```text
Crack #17

9/21  3.2mm
9/22  3.4mm
9/23  4.1mm
9/24  5.8mm ⚠
```

서비스는 단순 Threshold에서 발전할 수 있다.

```text
정상
↓
변화량 증가
↓
추세 이상
↓
Alert
```

이 부분에 장기 이상·특이징후 탐지 모델을 적용할 수 있다.

---

# 32. Week 5 — 2.5D

여기서 Three.js를 추가한다.

새 시스템을 만드는 것이 아니라 기존 Spatial Core를 그대로 사용한다.

```text
Spatial Core
├─ Canvas 2D
└─ Three.js 2.5D
```

가장 현실적인 첫 2.5D는 Floor Exploded View다.

```text
        ┌─────────┐
        │   4F    │
        └─────────┘

     ┌──────────────┐
     │      3F      │ ← 🔴 균열
     └──────────────┘

   ┌─────────────────┐
   │       2F        │
   └─────────────────┘

┌──────────────────────┐
│          1F          │
└──────────────────────┘
```

이 단계에서 Full BIM Viewer를 목표로 하지 않는다.

핵심은 동일 Event/Spatial Data가 2D와 2.5D에서 모두 동작하는 것을 보여주는 것이다.

---

# 33. Week 5까지가 본제품

Week 5 기준으로 다음이 완성되어야 한다.

```text
도면 입력
+
공간 좌표
+
AI 탐지
+
이벤트
+
시간축
+
관제
+
2D
+
2.5D
```

이 상태만으로도 프로젝트 설명이 완결돼야 한다.

Unreal이 없어도 프로젝트가 실패한 것으로 보이면 안 된다.

---

# 34. Phase 3 — Expansion Pack
## Week 6~7

이 단계부터는 실패해도 본제품이 살아 있어야 한다.

Expansion 후보:

1. Unreal
2. AI 장기 이상탐지
3. Before/After Comparison
4. 기타 고도화

---

# 35. Unreal Expansion의 최소 범위

Unreal 전체 이식은 금지한다.

최초 Unreal PoC는 다음만 구현한다.

```text
Project 선택
↓
Floor 선택
↓
Spatial API 호출
↓
Unreal Scene
↓
Event 위치 표시
```

Web과 연결하면:

```text
2D Viewer
↓
Crack #18 선택
↓
Immersive View
↓
Unreal
↓
3F 이동
↓
같은 객체 위치 Focus
↓
Highlight
```

이것만 되어도 다음을 증명할 수 있다.

> **동일한 Digital Twin Core를 Web과 Unreal이 동시에 소비한다.**

---

# 36. Unreal Expansion의 초기 목표는 그래픽 품질이 아니다

처음부터 다음을 모두 목표로 하지 않는다.

- Nanite 극한 활용
- Lumen 최적화
- 초고품질 Asset
- Full BIM
- VR
- Multiplayer
- Physics Simulation
- 대형 Open World

초기에는:

```text
Simple Building Shell
+
Floor
+
Marker
+
API
+
Camera Navigation
```

정도로 충분하다.

첫 Demo Story:

1. Web에서 3층 균열 선택
2. `3D 현장 보기`
3. Unreal 열기
4. 동일 Object ID 전달
5. 동일 Spatial Position 이동
6. Highlight
7. Evidence/Event 표시

이것으로 2D → 2.5D → Immersive 3D 스토리가 완성된다.

---

# 37. AI Expansion

Unreal보다 제품 가치 측면에서 AI 고도화가 더 중요할 수도 있다.

기본 Detection:

```text
현재 균열 있음
```

고도화:

```text
과거 대비 균열 확대
```

더 나아가:

```text
과거 30일
↓
Baseline
↓
정상 변화 패턴
↓
현재 변화
↓
Anomaly Score
```

예:

```text
현재 상태

균열 폭          5.8 mm
7일 평균         3.6 mm
변화율           +61%
Anomaly score    0.91

⚠ 비정상 변화 가능성
```

이 방향은 프로젝트의 AI 정체성을 강화한다.

---

# 38. Before / After Expansion

시간이 허용하면 효과적인 시연 요소다.

```text
09/10       09/24

┌─────┐     ┌─────┐
│     │     │  🔴 │
│     │ →   │     │
└─────┘     └─────┘
```

또는:

```text
PAST │████████────│ CURRENT
```

형태의 Slider를 사용할 수 있다.

---

# 39. Phase 4 — Stabilization
## Week 8

Week 8에는 신규 기능 개발을 사실상 금지한다.

목표:

- Bugfix
- 통합 안정화
- 예외 처리
- Loading
- Error UX
- 성능 측정
- API 오류 처리
- 데이터 정합
- Demo Data
- 발표 Scenario
- 시연 영상
- 문서
- 최종 디자인 마감
- 장애 대비

이 프로젝트는 다음이 모두 연결된다.

```text
AI
Backend
Frontend
Spatial Data
2D
2.5D
Optional Unreal
```

따라서 통합에서 문제가 발생할 가능성이 높다.

마지막 주까지 기능 개발을 지속하면 위험하다.

---

# 40. 8주 권장 일정

## Week 1 — Foundation

```text
요구사항 Freeze
Domain Model
Spatial Model
Coordinate System
API Contract
Drawing Viewer
```

---

## Week 2 — Vertical Slice MVP

```text
AI
↓
Backend
↓
Event
↓
2D Map
↓
Detail
```

첫 번째 완제품 상태.

---

## Week 3 — Monitoring

```text
실시간 Event
Filter
Zone
Severity
State Management
Monitoring UI
```

---

## Week 4 — Temporal Twin

```text
History
Timeline
Trend
Anomaly
Before / After
```

---

## Week 5 — 2.5D

```text
Three.js
Floor
Object
Event Overlay
2D ↔ 2.5D
```

여기서 핵심 제품 완성.

---

## Week 6 — Expansion

상태에 따라 선택:

```text
Unreal PoC
```

또는

```text
AI 장기 이상탐지
```

---

## Week 7 — Expansion Integration

```text
Unreal ← Spatial API
```

또는:

```text
AI Anomaly Pipeline 강화
```

동시에 본제품의 완성도를 높인다.

---

## Week 8 — Feature Freeze

```text
Bugfix
Performance
UX
Integration
Demo
Presentation
Documentation
```

---

# 41. 단계별 중단 가능성

프로젝트는 어느 단계에서 일정이 종료돼도 결과물이 남아야 한다.

```text
Week 2 종료
→ Core MVP 존재

Week 5 종료
→ Core Product 존재

Week 7 종료
→ Expansion 포함 고도화 제품
```

피해야 할 구조:

```text
8주 동안 모든 기능 병렬 개발
↓
7주차 첫 통합
↓
연동 실패
↓
전체 기능 미완성
```

따라서 Dependency Chain을 짧게 유지한다.

---

# 42. Unreal Go / No-Go Gate

Unreal 도입 여부는 Week 5 말에 결정한다.

다음 조건을 확인한다.

- 2D Viewer 안정
- 2.5D Viewer 안정
- Spatial Model 안정
- 좌표 변환 안정
- AI → Event Pipeline 안정
- Timeline 안정
- 핵심 Demo Flow 완성
- Backend 안정
- 팀원의 필수 업무 미지연
- 주요 Blocking Issue 없음

대부분 만족하면:

> GO — Unreal Expansion 진행

핵심 항목이 흔들리면:

> NO-GO — Unreal 제외

대신 다음을 강화한다.

- AI Anomaly
- 2.5D
- UX
- Timeline
- Before/After
- 성능
- Demo 완성도

Unreal을 넣지 않아도 실패가 아니다.

---

# 43. 전체 제품 Layer

```text
L0 — Data
BIM / Drawing / CCTV / Sensor

        ↓

L1 — Intelligence
Detection / Tracking / Anomaly

        ↓

L2 — Digital Twin Core
Spatial / Event / Timeline / State

        ↓

L3 — Core Product
2D Monitoring

        ↓

L4 — Advanced View
Three.js 2.5D

        ↓

L5 — Expansion
Unreal / VR / Simulation
```

원칙:

> **아래 Layer가 완성되지 않았으면 위 Layer로 올라가지 않는다.**

---

# 44. 프로젝트 차별화 포인트

단순한 다음 프로젝트가 되어서는 안 된다.

```text
YOLO
+
Dashboard
```

대신 핵심 구조는 다음과 같아야 한다.

```text
Detection
↓
Spatial Mapping
↓
Temporal State
↓
Anomaly
↓
Situation Monitoring
```

즉 프로젝트의 본질은:

> AI가 무엇을 발견했는가

뿐만 아니라:

> 어디에서 발생했는가  
> 과거와 비교해 어떻게 변했는가  
> 지금 관리자가 봐야 하는 상황인가

를 판단할 수 있게 하는 것이다.

---

# 45. 프로젝트에서 Unreal을 설명하는 방법

Unreal을 다음과 같이 설명하지 않는다.

> 우리 프로젝트는 Unreal 기반 Digital Twin이다.

현재 구조에서는 과장에 가깝다.

대신:

> **핵심 제품은 Web 기반 2D/2.5D Spatial Monitoring System이며, 동일 Digital Twin Core를 소비하는 Unreal 기반 Immersive Expansion을 선택적으로 지원한다.**

라고 설명하는 것이 정확하다.

---

# 46. 상용 제품과의 경쟁 방식

기존 상용 제품을 다시 만들 필요는 없다.

목표:

```text
Revit을 새로 만들지 않는다.
Navisworks를 새로 만들지 않는다.
Cupix를 새로 만들지 않는다.
ACC를 새로 만들지 않는다.
```

대신 이들이 생성하거나 관리하는 데이터를 받을 수 있는 방향을 유지한다.

```text
BIM / Drawing
↓
Spatial Core

Reality Capture
↓
Spatial Core

CCTV / AI
↓
Event

Sensor
↓
State / Timeline
```

그 위에서 프로젝트만의 판단 및 관제 경험을 구현한다.

---

# 47. 최종 설계 원칙

## 원칙 1

**2D/2.5D가 본제품이다.**

## 원칙 2

**Unreal은 Expansion Pack이다.**

## 원칙 3

**Viewer를 데이터의 정본으로 만들지 않는다.**

## 원칙 4

**Spatial Core를 정본으로 한다.**

## 원칙 5

**Screen Coordinate를 저장하지 않는다. Site Coordinate를 저장한다.**

## 원칙 6

**Canvas / Three.js / Unreal은 같은 공간 데이터를 소비한다.**

## 원칙 7

**2D → 2.5D → Unreal 전환 시 Floor / Object / Position / Time Context를 유지한다.**

## 원칙 8

**Unreal 때문에 기존 Web Architecture를 뒤엎지 않는다.**

## 원칙 9

**Week 5까지 Unreal 없이 제품이 완성되어야 한다.**

## 원칙 10

**Unreal은 명확한 Go / No-Go Gate 이후에만 착수한다.**

## 원칙 11

**Week 8은 Feature Freeze다.**

## 원칙 12

**하위 계층이 완성되기 전에 상위 계층으로 올라가지 않는다.**

---

# 48. 최종 프로젝트 진행 정의

전체 8주 계획을 한 문장으로 표현하면 다음과 같다.

> **2주차에 실제 동작하는 건설 상황 관제 MVP를 만들고, 5주차까지 2D/2.5D와 시간축·공간 모델을 갖춘 핵심 Digital Twin 제품을 완성한 뒤, 6~7주차에 프로젝트 상황에 따라 AI 장기 이상탐지 또는 Unreal Immersive Expansion을 추가하고, 마지막 8주차는 기능을 동결한 채 통합·성능·UX·시연·문서 완성도에 집중한다.**

---

# 49. 최종 시스템 개념도

```text
┌───────────────────────────────────────────────────────────┐
│                        FIELD                              │
│                                                           │
│ CCTV     360 Camera     Drone     IoT     BIM / Drawing   │
└────┬─────────┬────────────┬────────┬───────────┬──────────┘
     │         │            │        │           │
     └─────────┴────────────┴────────┴───────────┘
                          │
                          ▼
               ┌────────────────────┐
               │   Data Ingestion   │
               └─────────┬──────────┘
                         │
                         ▼
               ┌────────────────────┐
               │ AI / Vision Layer  │
               │                    │
               │ Detection          │
               │ Tracking           │
               │ Change             │
               │ Anomaly            │
               └─────────┬──────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  DIGITAL TWIN CORE   │
              │                      │
              │ Project              │
              │ Building             │
              │ Floor                │
              │ Drawing              │
              │ Zone                 │
              │ SpatialObject        │
              │ Event                │
              │ State                │
              │ Timeline             │
              │ Coordinate Mapping   │
              └──────────┬───────────┘
                         │
            ┌────────────┼──────────────┐
            │            │              │
            ▼            ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │ Canvas/SVG │ │ Three.js   │ │  Unreal    │
    │            │ │            │ │            │
    │ 2D Core    │ │ 2.5D View  │ │ Expansion  │
    │ Monitoring │ │            │ │ Immersive  │
    └────────────┘ └────────────┘ └────────────┘

         CORE PRODUCT              OPTIONAL EXPANSION
```

---

# 50. 최종 판단

현재 8주 프로젝트에서 가장 위험한 선택은 Unreal을 조기에 핵심 의존성으로 만드는 것이다.

가장 안정적이고 확장 가능한 선택은:

```text
2D Monitoring
↓
Spatial / Temporal Digital Twin Core
↓
Three.js 2.5D
↓
Optional Unreal
```

순서다.

이 구조는 짧은 프로젝트 기간에도 MVP를 빠르게 확보할 수 있고, 일정이 허용될 경우 상당히 고급스러운 3D/Simulation까지 확장할 수 있다.

동시에 특정 렌더링 기술에 프로젝트 전체가 종속되는 문제를 피한다.

따라서 SENTIA의 시각화 전략은 최종적으로 다음과 같이 정의한다.

> **SENTIA의 핵심은 Unreal이나 Three.js가 아니라, 현장의 시간·공간·이벤트 상태를 통합하는 Digital Twin Core다. Canvas, Three.js, Unreal은 그 동일한 상태를 서로 다른 깊이로 표현하는 Viewer이며, 2D/2.5D가 기본 제품, Unreal은 선택형 고급 확장팩이다.**
