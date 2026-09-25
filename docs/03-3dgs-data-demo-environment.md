# SENTIA — 3D Gaussian Splatting · 현장 데이터 · 학습 · 시연환경 세션 정본

> 문서 목적: 본 세션에서 논의된 내용을 누락 없이 하나의 정본 문서로 통합한다.  
> 범위: 3D Gaussian Splatting(3DGS)과 Unreal Engine의 관계, 건설 현장 데이터 수집/가공/학습, 공개 데이터·오픈소스 활용, 최소 장비 구성, CCTV 기반 공간화, 3DGS 확장 전략, 실제 시연/검증 환경 구성.  
> 프로젝트 맥락: SENTIA — 건설 현장 안전·상태 관제를 중심으로 한 AI/Spatial/Digital Twin 시스템. 코어는 2D/2.5D 관제와 AI 파이프라인이며 Unreal + 3DGS는 후반 확장팩으로 취급한다.

---

# 0. 세션 전체 결론

본 세션을 통해 다음 방향이 정리되었다.

1. **3DGS는 Unreal의 대체물이 아니라 Reality Layer다.**
   - 3DGS: 실제 공간 외관 재현
   - Mesh/Point Cloud: 기하·충돌·측정
   - BIM/도면: 의미·구조·구역
   - Unreal: 이들을 통합해 시각화·상호작용하는 실행 환경

2. **고정 CCTV 1대만으로 3DGS를 만드는 것은 현실적이지 않다.**
   - 3DGS는 서로 다른 위치에서 확보한 multi-view image가 필요하다.
   - 단일 CCTV는 실시간 안전 관제용으로 사용한다.

3. **그러나 실제 공사현장에 3DGS용 이동 카메라를 상시 운용하는 것도 비현실적이다.**
   - 상시 운영은 고정 CCTV 1대를 코어로 둔다.
   - 3DGS는 관리자/점검자의 기존 site walk 때 스마트폰으로 짧게 촬영하는 **주기적 snapshot acquisition**으로 처리한다.

4. **초기 장비 최소안은 사실상 다음으로 줄일 수 있다.**
   - 고정 CCTV 1대
   - 일반 스마트폰 1대
   - 도면
   - 기준점 4~6개
   - 추가 전용 3D 스캐너, 360 카메라, LiDAR, 다중 CCTV는 MVP 필수가 아니다.

5. **CCTV 1대에서도 단순 PPE 검출보다 훨씬 많은 정보를 추출한다.**
   - Detection
   - Tracking
   - Homography
   - World XY
   - Zone
   - Trajectory
   - Speed
   - Dwell time
   - Risk relationship
   - Event queue

6. **정확한 3D XYZ를 처음부터 얻으려 하지 않는다.**
   - 실시간 관제는 `floor + (x, y)` 기반 2.5D로 시작한다.
   - Z는 floor elevation 등 정적 구조정보에서 보충한다.

7. **학습 데이터는 공개 데이터로 bootstrap하고, 실제 사용할 카메라 데이터로 domain adaptation한다.**
   - AI-Hub 건설 데이터
   - SARD
   - PPE 공개 데이터
   - Worker Safety Twin 등의 오픈소스 구현
   - 자체 CAM01 영상
   - active learning / hard-negative mining

8. **CCTV를 모든 문제의 센서로 사용하지 않는다.**
   - PPE/작업자/장비 동선 → CCTV
   - 균열 → 근접 고해상도 이미지/드론/inspection camera
   - 장비 위치/가동 → 가능하면 GPS/GNSS/telematics
   - 화재/과열 → 필요 시 thermal
   - 공정 상태 → periodic reality capture

9. **실제 공사현장을 최종 시연장으로 의존하지 않는다.**
   - Level 1: SSAFY 내부 모의현장 → 라이브 데모
   - Level 2: 안전체험시설/산업형 공간 → 외부 도메인 검증
   - Level 3: 실제 건설 공개 데이터/가능하면 실제 현장 1회 → generalization evidence

10. **최종 제품의 기술축은 ‘AI 모델 하나’가 아니다.**
    - `pixel → track → world XY → zone → temporal context → risk → event`
    - 이 중간 Spatial/Temporal/Risk Pipeline이 SENTIA의 핵심 차별점이다.

---

# 1. 3D Gaussian Splatting과 Unreal Engine의 관계

## 1.1 기본 정의

3D Gaussian Splatting(3DGS)은 Unreal Engine 자체의 기능이 아니다.

정확한 관계는 다음과 같다.

```text
3DGS
= 현실 공간을 사진/영상 기반으로 실사형 3D 표현으로 만드는 기술

Unreal Engine
= 3DGS + Mesh + BIM + AI Event + UI를 결합하는 실행/시각화 환경
```

보다 역할을 명확히 나누면:

```text
3DGS = 현실 공간의 실사 껍질 / Reality Layer
BIM / Mesh = 구조·기하·객체 의미
Unreal = 이들을 합치고 움직이며 상호작용시키는 Runtime
```

3DGS는 전통적인 triangle mesh 대신 다수의 3D Gaussian primitive로 장면을 표현한다.

각 Gaussian은 대략 다음 속성을 가진다.

- 위치
- 크기
- 방향
- 색
- 투명도
- view-dependent appearance 관련 파라미터

초기 3DGS 연구는 고품질 novel-view synthesis와 실시간 렌더링을 핵심 장점으로 제시했다.

참고:
- https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/

---

# 2. Unreal에서의 활용 구조

대표적인 Unreal 통합 구조는 다음과 같다.

```text
현장 촬영
   ↓
3DGS Reconstruction
   ↓
PLY / SPZ / plugin-specific format
   ↓
Unreal Plugin
   ↓
UE Level
```

Unreal 내부에서는 아래 요소가 공존할 수 있다.

```text
┌──────────── Unreal Engine ────────────┐
│                                      │
│  3DGS Reality Layer                  │
│          +                           │
│  BIM / CAD / Proxy Mesh              │
│          +                           │
│  AI Detection / Event                │
│          +                           │
│  HUD / Timeline / Interaction        │
│                                      │
└──────────────────────────────────────┘
```

조사 중 언급된 Unreal 연동 플러그인/프로젝트:

- DazaiStudio Splat Renderer
- XGRIDS LCC 3DGS Unreal Plugin
- Luma Unreal Plugin
- Yandex YAGS 계열
- Fab의 여러 3DGS rendering plugin

참고:
- https://github.com/DazaiStudio/SplatRenderer-UEPlugin
- https://github.com/xgrids/LCC-3DGS-Unreal-Plugin
- https://www.fab.com/listings/b52460e0-3ace-465e-a378-495a5531e318
- https://github.com/yandex/yags

---

# 3. 건설 디지털 트윈에서의 3DGS 위치

이번 SENTIA 구조에서는 다음처럼 보는 것이 가장 적절하다.

| 계층 | 역할 |
|---|---|
| 2D/2.5D 도면 | 현황 파악, 위치, 경량 관제 |
| BIM / CAD | 구조·객체 의미·정적 geometry |
| Mesh / Point Cloud | collision, raycast, measurement |
| 3DGS | 실제 현장의 최신 시각 상태 |
| AI Vision | PPE, 작업자, 장비, 위험상황 |
| Backend | 시간·상태·이벤트 |
| Unreal | 통합 시각화 및 3D 확장 |

예시 UX:

```text
2D 평면도
   ↓
3층 A구역 위험 이벤트 클릭
   ↓
해당 위치의 Unreal 3D View
   ↓
최근 3DGS Reality Snapshot
   +
AI Hazard Marker
   +
과거 시점 Timeline
```

핵심:

> 3DGS를 BIM/Mesh 대체재로 쓰지 않는다.

3DGS는 시각적 현실감에는 강하지만 다음 부분은 약하다.

- semantic object ID
- 정확한 구조 정보
- collision
- physics
- navmesh
- 정밀 측정
- 객체 단위 편집
- 구조적 reasoning

따라서 권장 구조:

```text
보이는 것      = 3DGS
계산하는 것    = BIM / Mesh / Point Cloud
운영하는 것    = Unreal / Backend
```

---

# 4. 프로젝트 단계에서의 위치

SENTIA는 Unreal/3DGS를 코어로 시작하지 않는다.

```text
Phase 1 — Core MVP
2D/2.5D
AI Event
Tracking
Spatial Mapping
Risk Queue
Dashboard

Phase 2 — Product Structure
World Coordinate
Floor / Zone
History
Timeline
Backend Event Contract

Phase 3 — Extension
Unreal
+
3DGS
+
BIM/Proxy Mesh
+
AI Event Replay
```

즉 3DGS가 없어도 제품이 완성되고, 넣었을 때 “Reality Layer”로 제품 가치가 올라가야 한다.

---

# 5. 최초 데이터 수집 논의와 문제점

초기에는 다음과 같은 데이터 취득 구조가 제안되었다.

```text
현실 현장
 ├ RGB 이동 촬영
 ├ LiDAR / RGB-D
 ├ 고정 CCTV 2~4
 ├ 360 Camera
 └ Control Points
```

목표:

```text
RGB → SfM → 3DGS
LiDAR → Metric Geometry
CCTV → Live AI
BIM → Semantic Layer
```

그러나 이에 대해 실제 공사현장 관점에서 중요한 문제가 제기되었다.

> 상식적으로 실제 공사 현장에서 3DGS 전용 이동 카메라가 계속 굴러다니는 운용구조는 현실성이 떨어진다.

이 지적에 따라 구조를 대폭 단순화했다.

---

# 6. 정정된 최소 장비 구조

최종적으로 최소 장비는 다음이 적절하다.

| 구성 | 수량 | 역할 |
|---|---:|---|
| 고정 CCTV | 1 | 상시 AI 관제 |
| 일반 스마트폰 | 1 | 주기적 3DGS Snapshot / 근접 inspection |
| 도면 | 기존 | World Coordinate 기준 |
| 기준점 | 4~6개 | 카메라/도면/3DGS 좌표 정합 |

즉 **추가 전용 장비 0대**도 가능하다.

선택적 확장:

- CCTV 2대째: occlusion/coverage 한계가 검증되면
- RGB-D/LiDAR: metric accuracy가 부족할 때
- Thermal: 화재/과열이 핵심 use case가 될 때
- GNSS/telematics: 장비 위치를 vision보다 정확히 얻을 수 있을 때
- 360 camera: 주기적 site capture가 실제 운영과 잘 맞을 때

---

# 7. CCTV 1대에서 확보해야 할 정보

단순 구조:

```text
CCTV
 ↓
YOLO
 ↓
Helmet Missing
 ↓
Alert
```

이 수준에서 끝내면 제품 차별성이 약하다.

권장 구조:

```text
Fixed CCTV
    ↓
Detection
    ↓
Tracking
    ↓
Pixel Position
    ↓
Ground Homography
    ↓
World XY
    ↓
Zone / Trajectory / Speed / Dwell
    ↓
Spatial + Temporal Rule
    ↓
Risk Event
```

## 7.1 작업자 위치

작업자의 ground contact approximation:

```text
┌──────────┐
│ person   │
│          │
│          │
└────●─────┘
     ↑
bbox bottom-center
```

이 pixel을 ground point로 사용한다.

현장/모의현장에 4개 이상의 기준점을 둔다.

```text
P1 ───────── P2
│             │
│             │
P3 ───────── P4
```

예:

```text
P1 = (0, 0)
P2 = (8, 0)
P3 = (0, 6)
P4 = (8, 6)
```

영상 좌표와 실제 floor 좌표 사이 Homography를 계산한다.

가능해지는 정보:

```text
Worker #13
position      = (5.2, 2.8)
zone          = Hazard-B
direction     = East
speed         = 1.2 m/s
PPE           = Helmet Missing
dwell_time    = 4.8 sec
```

참조 연구:
- Fixed CCTV + object detection + tracking + ground-plane homography 기반 construction worker localization
- https://www.mdpi.com/1424-8220/23/20/8371

해당 논문의 특정 실험에서 약 7cm 수준 오차가 보고되었지만, 이는 모든 환경에서 보장되는 값이 아니라 실험 조건에 종속된 참고값이다.

---

# 8. 실시간 관제는 2.5D로 시작

초기부터 단일 CCTV에서 정확한 XYZ를 요구하면 난도가 급격히 올라간다.

실제 위험관제에서 많은 이벤트는 다음 정보면 충분하다.

```text
Floor / Level
+
x, y
```

예:

- 위험구역 진입
- 장비 접근
- 사람↔장비 거리
- 동선
- 혼잡도
- 체류시간
- PPE 상태

따라서 데이터 모델:

```text
Building
 └ Floor
    └ Zone
       └ (x, y)
```

Z는 필요할 때:

```text
z = Floor elevation
```

으로 보충한다.

정리:

```text
실시간 AI 관제 = 2.5D
3DGS = 3D Reality Visualization
```

---

# 9. 3DGS의 실제 현장 데이터 취득 방식

3DGS는 운영 센서가 아니다.

**정기 점검/공정 기록 때 잠깐 수행하는 measurement/capture activity**로 취급한다.

```text
고정 CCTV → 24/7

정기 site walk
    ↓
관리자 스마트폰으로 핵심 Zone 촬영
    ↓
3DGS Snapshot
```

실제 reality-capture 상용 서비스도 기존 site walk에 capture를 얹는 접근을 사용한다.

예:
- OpenSpace Capture
- 주기적 현장 워크스루 기반 공정기록

참고:
- https://www.openspace.ai/products/capture/
- https://support.openspace.ai/hc/en-us/articles/41951421694099-OpenSpace-Best-Practices

---

# 10. 전체 현장을 3DGS로 만들 필요가 없다

MVP에서는 전체 현장 대신 **Inspection Cell / Demo Zone**만 만든다.

예:

```text
전체 현장
┌────────────────────────┐
│                        │
│       ┌──────────┐     │
│       │ Zone B   │     │
│       │ CCTV ●   │     │
│       └──────────┘     │
│                        │
└────────────────────────┘
```

대략:

- 5m × 8m
- 10m × 10m

정도의 핵심 구역만 3DGS로 만든다.

---

# 11. 3DGS 입력 촬영 원칙

고정 CCTV 한 위치에서 pan/tilt만 바꾸는 것은 정상적인 3DGS 입력이 아니다.

필요:

- 서로 다른 camera position
- 충분한 image overlap
- static scene
- blur 최소화
- 조명 급변 최소화
- moving object 최소화
- 대상이 여러 view에서 관측

참고:
- COLMAP: https://colmap.github.io/tutorial
- RealityScan photogrammetry camera movement
- Postshot capture guideline
- Nerfstudio custom dataset

권장 MVP 방식:

```text
관리자 스마트폰
 ↓
1~수분간 핵심 Zone을 걸으며 촬영
 ↓
Video → frame extraction
 ↓
COLMAP / Postshot
 ↓
3DGS
```

---

# 12. 3DGS 가공 파이프라인

권장 정본:

```text
RAW RGB Video
    ↓
Frame Extraction
    ↓
Blur / Near-Duplicate / Dynamic Filtering
    ↓
COLMAP SfM
    │
    ├──────────────┐
    │              │
    ▼              ▼
Camera Poses   Sparse Point Cloud
    │              │
    │              ▼
    │             MVS
    │              │
    │              ▼
    │         Dense Point Cloud
    │              │
    │              ▼
    │             Mesh
    │
    ▼
gsplat
    ↓
3DGS
```

동일한 COLMAP pose를 사용하면 Visual Layer와 Geometry Layer를 같은 좌표계에 둘 수 있다.

권장 툴:

### Reproducible Pipeline
- COLMAP
- gsplat

참고:
- https://colmap.github.io/
- https://docs.gsplat.studio/

### Fast Prototype
- Postshot

참고:
- https://www.jawset.com/docs/d/Postshot%2BUser%2BGuide

---

# 13. LiDAR에 대한 최종 판단

MVP 필수 아님.

초기 metric scale은:

- 도면
- control point
- 실제 거리

로 잡는다.

예:

```text
실제 A-B = 5.0m
SfM A'-B' = 2.743 unit

scale = 5 / 2.743
```

여러 기준점으로 similarity transform을 계산한다.

```text
3DGS/SfM
  ↓ scale + rotation + translation
Project World
```

LiDAR/RGB-D는 다음 조건일 때 확장한다.

- 위치 오차가 실제 제품 요구를 못 만족
- 구조물 geometry 정합이 중요한 기능으로 승격
- collision/measurement 검증이 필요

---

# 14. 3DGS Timeline 전략

이번 프로젝트에서 진짜 4DGS를 바로 넣지 않는다.

대신:

```text
T0 → site_000
T1 → site_001
T2 → site_002
T3 → site_003
```

같은 좌표계에 여러 3DGS Snapshot을 둔다.

Unreal:

```text
T0 ─── T1 ─── [T2] ─── T3
```

즉 **Snapshot Timeline**으로 4D-like UX를 만든다.

진짜 4DGS는 후속 연구영역으로 둔다.

참고:
- 4D Gaussian Splatting, CVPR 2024
- 4C4D, CVPR 2026
- Splat Labs construction progress 사례

---

# 15. 공개 학습 데이터 조사

## 15.1 SARD 2026

건설현장 activity recognition용 최신 공개 자료.

특징:

- 실제 construction environment
- 748 video clips
- 11 activity classes
- 27 human-object interaction categories
- fine-grained subset:
  - 9 long videos
  - 67,098 annotated frames
  - 1,018,232 person boxes
- 약 43.3GB
- Zenodo 공개
- 라이선스 표기가 명확하지 않은 상태이므로 재배포/상업 이용은 별도 확인 필요

용도:
- temporal model
- action recognition
- human-object interaction
- worker behavior understanding

참고:
- https://zenodo.org/records/19926448

---

# 16. ConstrID

실제 초고층 건설현장 CCTV 기반 worker ReID 데이터.

특징:

- 실제 가동 중인 construction site
- 4MP surveillance cameras
- 약 6개월
- 4,409 real images
- 112 real workers
- synthetic data 728
- 실제 construction activity 포함:
  - formwork
  - concrete pouring
  - rebar tying 등
- 신청형
- academic/education 중심
- commercial use / redistribution 제한

용도:
- multi-camera ReID
- worker identity persistence
- long-term tracking 연구

참고:
- https://github.com/Drlijiaqi/ConstrID-Dataset

---

# 17. AI-Hub — 공사현장 안전장비 인식 데이터

대규모 PPE bootstrap에 매우 유용.

규모:

- 약 203만 이미지
- bbox 약 9,481,123
- safety belt polygon 76,769
- keypoint 128,656
- JPG + JSON

용도:
- helmet
- shoes
- belt
- person
- PPE on/off

전략:

> PPE base model은 자체 촬영 수천 장부터 만들 필요가 없다.  
> AI-Hub로 construction-domain base를 만들고 자체 CAM01로 domain adaptation한다.

참고:
- https://aihub.or.kr/aihubdata/data/view.do?aihubDataSe=realm&currMenu=115&dataSetSn=163&topMenu=

---

# 18. AI-Hub — 고소작업 현장 실시간 영상

규모:

- 371시간
- 687,895 frames
- CCTV 3인칭: 548,479
- 1인칭: 나머지 약 20%

대상:

- helmet
- safety belt
- hook
- safety shoes
- scaffold
- aerial work vehicle
- railing
- ladder
- 위험 작업 행동

중요한 특성:

- 실제 사고 자연발생 데이터가 아니라
- 정상/위험상황 시나리오를 연출하여 제작한 데이터

따라서:

```text
실제 현장 정상영상
+
안전하게 연출된 공개 위험영상
```

구조로 사용하는 것이 적절하다.

참고:
- https://aihub.or.kr/aihubdata/data/view.do?dataSetSn=507&topMenu=100

---

# 19. AI-Hub — 건설 현장 위험 상태 판단

규모:

- 약 226,000 images
- bbox + segmentation

대상:
- 추락
- 낙하
- 끼임/충돌
- 화재
- 전도 등

중요한 설계 힌트:

공식 설명도 행동 자체를 하나의 end-to-end classifier로 판별하는 것보다 **위험 시나리오에 필요한 객체의 존재와 관계**를 이용한다.

따라서 SENTIA도:

```text
Object Detection
    ↓
Tracking
    ↓
Spatial Relationship
    ↓
Temporal Context
    ↓
Risk Engine
```

구조가 적절하다.

참고:
- https://aihub.or.kr/aihubdata/data/view.do?aihubDataSe=realm&currMenu=&dataSetSn=71407&topMenu=

---

# 20. Worker-Safety-Twin 오픈소스

본 프로젝트와 구조적으로 유사한 reference implementation.

구조:

```text
YOLOv8
 ↓
SORT
 ↓
Perspective Projection
 ↓
SQLite
 ↓
BIM Visualization
```

DB 예:

```text
person_id
cam_id
floor
datetime
x_location
y_location
classification
```

특징:

- 자체 object dataset 약 3,455 images
- worker / helmet / vest
- perspective projection
- BIM visualization
- 특정 실험에서 mean position error 약 13.2cm 보고

정확도 수치는 환경 종속이며 일반화 보장값이 아니다.

참고:
- https://github.com/almosenja/Worker-Safety-Twin

이 프로젝트는 SENTIA의 다음 구조에 직접 참고 가능:

```text
CCTV Pixel
  ↓
Track
  ↓
World XY
  ↓
BIM / Map
  ↓
Safety Event
```

---

# 21. PPE-Detection-for-Construction-Site-Safety

한국 팀이 실제로 여러 공개 source를 통합한 사례.

사용 데이터:

- AI-Hub PPE
- Roboflow PPE
- Hard Hat Workers
- vest dataset
- boots dataset

가공:
- CVAT
- label 정리
- class 통합

모델:
- YOLOv5
- YOLOv7
- DeepSORT

최종 7 classes:

```text
Safety_Belt
No_Safety_Belt
Safety_Shoes
No_Safety_Shoes
Safety_Helmet
No_Safety_Helmet
Person
```

규모:
- train 11,495 images
- validation 1,437
- train bbox 약 84,323

참고:
- https://github.com/NamHoKi/PPE-Detection-for-Construction-Site-Safety

의미:

> “여러 public construction/PPE dataset을 taxonomy 통합 후 재학습”은 이미 재현된 실전적 접근이다.

---

# 22. Tracking-and-Material-Counting

기존 site surveillance camera를 활용한 건설 생산성 분석 사례.

기능:
- worker detection
- material detection
- material classification/counting
- installed/imported/waste estimation
- worker count

환경:
- low-light
- indoor construction site

기술:
- YOLOv4
- DenseNet
- image processing

참고:
- https://github.com/smartconstructiongroup/Tracking-and-Material-Counting

의미:

고정 CCTV가 PPE만 보는 게 아니라 다음까지 확장 가능함을 보여준다.

```text
작업자 수
작업자 동선
장비/자재
자재 입출고
생산성
구역 체류
```

---

# 23. Construction-PPE / Atharion PPE

## Ultralytics Construction-PPE

- 1,416 images
- 11 classes
- construction scene
- YOLO ecosystem
- AGPL-3.0

용도:
- 빠른 baseline
- pipeline 검증

참고:
- Ultralytics construction PPE dataset

## Atharion PPE v1

- 1,130 images
- 5,355 boxes
- 라이선스 provenance 정리
- scene-group 단위 split

특히 중요한 점:

**같은 video 인접 frame을 train/validation으로 나누지 않음.**

SENTIA도 반드시 적용:

```text
Day/Session A,B → Train
Day/Session C   → Validation
Day/Session D   → Test
```

참고:
- https://github.com/atharion-team/ppe-dataset-v1

---

# 24. 일반 이상행동 CCTV 데이터

건설 최종평가용은 아니지만 temporal pretraining/pipeline 검증용으로 사용할 수 있다.

예:
- AI-Hub 이상행동 CCTV
- 지하철 CCTV

포함 행동 예:
- 실신
- 침입
- 배회
- 전도
- 계단/에스컬레이터 사고

사용 위치:

```text
Generic Temporal Representation
        ↓
Construction-specific Fine-tuning
```

단, 건설 데이터의 직접 대체재로 사용하면 안 된다.

---

# 25. 장비 데이터는 Vision만 고집하지 않는다

AI-Hub의 건설 현장 장비 모니터링·생산성 데이터는 다양한 modality를 포함한다.

예:

- 영상
- 이미지
- sensor
- location logger
- telematics

규모 예:

- 장비 위치 궤적: 3,870일
- 자재 입출고 영상: 312시간
- 자재 이미지: 101,459장
- 굴착기 데이터: 100,406
- 총량 약 1.58TB

참고:
- https://aihub.or.kr/aihubdata/data/view.do?aihubDataSe=realm&currMenu=&dataSetSn=71388&topMenu=

의미:

> 장비가 자체적으로 위치/가동정보를 제공하면 CCTV AI가 다시 추정할 필요가 없다.

권장 구조:

```text
Worker     → CCTV Vision
Equipment  → CCTV + Telematics
Floor/Zone → BIM / Drawing
Risk       → Sensor Fusion
```

---

# 26. 균열 데이터와 센서 전략

균열은 멀리 있는 CCTV에서 해결하려 하지 않는다.

AI-Hub SOC 시설물 균열 데이터:

- 약 525,000 images
- defect 10종
  - crack
  - network crack
  - delamination
  - spalling
  - efflorescence
  - leakage
  - exposed rebar
  - material segregation
  - lifting
  - damage
- polygon/polyline annotation

수집 방식:
- DSLR
- drone
- scanning

즉 균열은 근접/고해상도/inspection sensor가 적합하다.

권장:

```text
Inspection Photo
    ↓
Crack Segmentation
    ↓
length / area / type
    ↓
Structure / Zone
    ↓
Defect Event
```

참고:
- AI-Hub SOC 균열 데이터
- METU Concrete Crack Images
- DeepCrack

METU:
- 고해상도 concrete images
- 40,000 patch
- crack 20,000 / no-crack 20,000
- CC BY 4.0

DeepCrack:
- segmentation benchmark
- non-commercial research/education restrictions 확인 필요

---

# 27. 화재/과열 데이터 전략

AI-Hub 무인 플랜트 안전 감시 데이터:

Video:
- RGB 13,400
- Thermal 6,600

Image:
- RGB 268,000
- Thermal 132,000

대상:
- intrusion
- leakage
- damage
- corrosion
- overheat
- PPE
- fire

의미:

화재/과열이 SENTIA의 중요 use case가 되면 RGB 모델을 과도하게 복잡하게 하기보다 thermal sensor 추가가 더 합리적일 수 있다.

추가 공개 자료:
- D-Fire Dataset
  - 21,000+ images
  - fire bbox 14,692
  - smoke bbox 11,865
  - YOLO annotations
  - surveillance video
  - pretrained models

참고:
- https://github.com/gaia-solutions-on-demand/DFireDataset

---

# 28. Wearable / IMU 가능성

VTT-ConIoT:

- 실제 전문 construction environment
- 13 workers
- 16 activities
- 3개 body position의 inertial sensor
- 보조 video / pose keypoint

의미:

장기적으로:

```text
Camera
+
Wearable IMU
```

구조도 가능하다.

그러나 이번 MVP에서는 작업자 착용 강제라는 운영비용 때문에 제외한다.

참고:
- https://zenodo.org/records/4683703

---

# 29. Synthetic 데이터

SynthSite:

- 2026 공개
- construction safety video
- 약 227 video
- 115 unsafe
- 112 safe
- suspended load 위험 시나리오 등
- 생성형 video 모델 활용

적절한 용도:

```text
Synthetic → Train Augmentation   O
Synthetic → Final Test           X
```

실제 카메라 노이즈, 조명, 먼지, occlusion, perspective, installation angle을 완전히 대체할 수 없다.

참고:
- https://huggingface.co/datasets/govtech/SynthSite

---

# 30. 공개 데이터 활용의 최종 분류

| 데이터 | 주 역할 |
|---|---|
| AI-Hub PPE | PPE base model |
| AI-Hub 고소작업 | PPE + pose + 위험상황 |
| AI-Hub 위험상태 | 위험 객체/관계 |
| SARD | temporal activity |
| ConstrID | ReID / multi-camera |
| Worker-Safety-Twin | spatial/BIM architecture |
| NamHoKi PPE | dataset merge/label 사례 |
| AI-Hub equipment | equipment + telemetry |
| AI-Hub SOC crack | defect |
| METU / DeepCrack | crack |
| AI-Hub Plant | thermal/fire |
| D-Fire | fire/smoke |
| VTT-ConIoT | wearable activity |
| SynthSite | rare-event augmentation |

---

# 31. 라이선스 관리 원칙

각 공개 dataset은 다운로드 가능 여부와 실제 사용/재배포 가능 범위를 구분해야 한다.

예:

- ConstrID: redistribution / commercial 제한
- Ultralytics Construction-PPE: AGPL-3.0
- METU Crack: CC BY 4.0
- SARD: 현재 record의 license 표시가 명확하지 않으므로 별도 확인
- AI-Hub: 신청형 및 개별 이용정책 확인

따라서:

```text
Git Repository
  ≠
Raw Dataset Storage
```

원본 공개 데이터는 저장소에 포함하지 않고 다운로드 manifest / script / metadata만 관리하는 것이 적절하다.

---

# 32. 데이터 디렉터리 구조

권장:

```text
data/
│
├── 00_public/
│   ├── ppe/
│   ├── activity/
│   ├── equipment/
│   ├── crack/
│   └── fire/
│
├── 10_site_raw/
│   └── CAM01/
│
├── 20_site_selected/
│
├── 30_annotations/
│
├── 40_train/
│
├── 50_validation/
│
├── 60_test/
│
└── 90_reality_capture/
    ├── T000/
    ├── T001/
    └── T002/
```

핵심:

- public / own-site 물리 분리
- license provenance 관리
- test contamination 방지
- 3DGS Reality Capture는 AI training corpus와 분리

---

# 33. 학습 전략

처음부터 자체 데이터만으로 만들지 않는다.

```text
Public Construction Dataset
      ↓
Construction-domain Base Model
      ↓
Own CAM01 Raw Video
      ↓
Baseline Inference
      ↓
FP / FN / Low Confidence 수집
      ↓
Human Labeling
      ↓
Site Fine-tuning
      ↓
Hold-out Site Test
      ↓
Deployment
      ↓
Hard-negative Mining
      ↓
Next Iteration
```

즉 Active Learning Loop.

---

# 34. 자체 현장 데이터 수집 목표

공개 데이터가 general construction appearance를 담당한다.

자체 데이터는 **“우리 카메라의 domain”**을 학습하기 위한 것.

초기 engineering target:

| 데이터 | 목표 |
|---|---:|
| raw CCTV | 3~7개 날짜/세션 |
| detector 후보 | 2,000~5,000 diverse frames |
| 실제 수작업 라벨 | 1,000~3,000 frames부터 |
| tracking test | 10~30 continuous clips |
| clip length | 약 10~60 sec |
| hold-out test | 최소 1일/1세션 완전 분리 |
| calibration | 기준점 4개 이상 |
| hard negatives | 적극 수집 |

위 수치는 공식 표준이 아니라 프로젝트 규모에 맞춘 시작점이다.

---

# 35. Hard Negative의 중요성

실제 서비스에서 중요한 문제:

- false positive
- alert fatigue
- “양치기 소년” 현상

따라서 정상/오탐 유발 scene을 적극 수집한다.

예:

```text
사람 없음
정상 작업
PPE 정상
반사
그림자
조명 변화
부분 가림
작업자 겹침
장비가 사람 가림
먼 거리 person
자재 이동
유사색 object
```

실제 제품에서는 “공개 위험영상 100만 장”보다 “우리 CAM01에서 계속 오탐나는 500장”이 더 가치 있을 수 있다.

---

# 36. 위험행동 자체 촬영 원칙

자체 환경에서 실제 위험사고를 재현하지 않는다.

가능:

- helmet on/off
- vest on/off
- walking
- sitting
- standing
- 가상 위험구역 진입/이탈
- equipment proxy 접근
- occlusion
- crowd

하지 않는 것:

- 실제 추락
- 실제 충돌
- 낙하물 사고
- 위험한 전동공구 운용

희귀 고위험 event는:

```text
공개 위험상황 데이터
+
Synthetic augmentation
+
Object relationship
+
Rule Engine
```

으로 대응한다.

---

# 37. 모델 구조 — 하나의 만능 모델 금지

권장:

```text
CAM01
 │
 ├ Person/PPE Detector
 ├ Equipment Detector
 ├ Tracker
 └ Pose/Temporal (필요시)
       │
       ▼
Observation
       │
       ▼
Spatial / Temporal Engine
       │
       ├ position
       ├ PPE state
       ├ speed
       ├ trajectory
       ├ zone
       └ duration
       │
       ▼
Risk Engine
       │
       ▼
Event
```

균열:

```text
Inspection Image
   ↓
Crack Segmentation
   ↓
Defect Event
```

화재:

```text
RGB / Thermal
   ↓
Fire / Smoke / Heat
   ↓
Event
```

마지막 **Event Schema에서 통합**한다.

---

# 38. Annotation 내부 표준

| Task | 정본 형식 |
|---|---|
| Object/PPE | COCO-style bbox/segmentation |
| Model export | YOLO |
| Tracking | MOT-style + track ID |
| Pose | COCO keypoint |
| Temporal Action | clip start/end + class |
| Crack | polygon/mask |
| Camera | intrinsic/extrinsic/homography |
| Zone | polygon / GeoJSON 계열 |
| Sensor | timestamp + xyz/value |
| 3DGS | scene + camera pose |

원칙:

> Dataset을 모델 포맷에 종속시키지 않는다.

---

# 39. Backend Event Schema 방향

AI가 Unreal에 직접 의존하지 않는다.

```text
AI Pipeline
    ↓
Event Normalizer
    ↓
Backend
    ↓
Web / Unreal
```

예:

```json
{
  "eventId": "EVT-00321",
  "timestamp": "2026-10-21T14:32:15.302",
  "cameraId": "CAM-02",
  "eventType": "PPE_HELMET_MISSING",
  "confidence": 0.94,
  "target": {
    "trackId": 28,
    "type": "worker"
  },
  "position": {
    "x": 3.82,
    "y": 1.44,
    "z": 0.0
  },
  "zoneId": "ZONE-B",
  "evidence": {
    "frame": "...",
    "bbox": [812, 244, 935, 663]
  },
  "scene": {
    "snapshotId": "SITE-2026-10-21"
  }
}
```

이렇게 하면 Detector 교체가 Unreal/Backend를 깨지 않는다.

---

# 40. Unreal 내 레이어 책임

```text
Unreal
 │
 ├ 3DGS
 │   └ visual / reality
 │
 ├ Proxy Mesh
 │   └ collision / raycast
 │
 ├ BIM / Zone
 │   └ semantics
 │
 └ Event Actor
     └ worker / hazard / alert
```

원칙:

```text
Splat = Visual
Mesh = Physics
BIM = Semantic
Event = Dynamic State
```

---

# 41. 실제 시연환경 문제

실제 공사현장을 그대로 시연 환경으로 쓰는 것은 현실적으로 어려움이 크다.

문제:

- 출입 허가
- 안전교육
- 촬영 승인
- 개인정보
- 현장 작업자 동의
- 기업 보안
- 공정 일정
- 현장소장 승인
- 사고 재현 불가
- 프로젝트 일정 종속

따라서 실제 공사현장을 **최종 라이브 시연의 필수조건으로 두지 않는다.**

---

# 42. SSAFY 내부 Micro Construction Zone

SSAFY 내부 프로젝트실/대강의실 일부를 이용해 작은 안전한 모의현장을 만든다.

권장 크기:

```text
약 4m × 5m
~ 4m × 6m
```

구성:

- 안전모 2~3
- 안전조끼
- 라바콘
- 경고 tape
- 빈 box
- PVC pipe / lightweight material
- 이동식 cart
- floor marking
- fixed camera 1대

실제 위험물은 사용하지 않는다.

제외:

- 실제 비계
- 실제 구멍
- 무거운 자재
- 전동공구
- 추락장치
- 위험한 낙하물

개구부는 예를 들어:

```text
검은 매트
+
DANGER NO ENTRY
+
Hazard Polygon
```

으로 표현한다.

---

# 43. SSAFY Live Demo 시나리오

시연의 목적은 “공사장 연극”이 아니라 시스템 로직 검증이다.

예:

```text
1. Worker enters scene
     ↓
Person detected
     ↓
Track #17

2. 정상구역 이동
     ↓
PPE = OK

3. Hazard Zone 접근
     ↓
distance threshold
     ↓
WATCH

4. Zone 진입
     ↓
Risk Queue

5. Helmet 제거
     ↓
PPE state = Missing

6. 일정 시간 지속
     ↓
ALERT

7. Evidence clip / map / timeline
```

이렇게 하면 SSAFY 환경에서도 **실제 제품 로직**을 검증할 수 있다.

---

# 44. 외부 검증 장소 — 산업안전체험교육장

추천 외부 검증 루트:

- 안전보건공단 산업안전체험교육장
- 인천 등 수도권 접근 가능한 교육장

장점:

- 실제 산업/건설 안전 시뮬레이션 환경
- 비계/난간/사다리/가설 구조물 등 현실적인 background
- 실제 건설현장보다 안전
- 실제 위험사고 재현 필요 없음
- public training facility 형태

검증 목적:

```text
SSAFY
= clean indoor domain

Safety Training Facility
= industrial visual domain
```

확인:

- person recall
- helmet recall
- PPE state
- tracking persistence
- occlusion recovery
- false positives
- world-coordinate error
- zone event

주의:

> 교육 신청 가능 여부와 개인 카메라/AI 실험 촬영 허가는 별개이므로 사전 승인 필요.

---

# 45. 민간 안전체험장

안전보건공단 인정 민간 안전체험시설도 대안.

가능 유형:

- 기업 안전체험관
- 교육원
- 산업안전 훈련장

장점:
- construction-like background
- 실제 사고 위험 없이 validation 가능

단점:
- 외부 연구 촬영 허가 불확실
- 개별 기관 협의 필요

SSAFY/삼성 연계가 가능한 경우 삼성 계열 교육시설에 문의하는 것도 후보이나, 일정의 필수 전제로 두지는 않는다.

---

# 46. 산업형 촬영 공간 / 창고형 공간

외부 안전체험장이 어렵다면 차선책.

필요한 visual characteristic:

- concrete
- steel
- box/pallet
- column
- wide floor
- industrial lighting
- clutter
- partial occlusion

후보:
- 산업형 촬영 스튜디오
- 빈 창고
- 물류창고형 studio
- 공장형 location
- 빈 상가/업무공간

장점:

> 처음부터 “촬영 공간”을 빌리는 것이므로 실제 공사현장보다 촬영/보안/안전 리스크가 낮다.

---

# 47. KICT / 스마트건설 실증 인프라

후속 단계 후보.

예:
- 한국건설기술연구원 연천 SOC 실증센터
- 스마트건설지원센터
- 국토부 스마트건설 기술 실증지원사업

특징:
- 실제 construction R&D testbed
- 실규모 SOC 시설
- 기업/연구기관 기술 실증 지원

이번 8주 프로젝트의 필수경로가 아니라:

```text
SSAFY 프로젝트
    ↓
성과
    ↓
포트폴리오 / 후속 연구 / 창업
    ↓
실증센터
    ↓
실제 현장
```

순서로 접근한다.

---

# 48. 실제 공사현장 직접 확보 루트

가능하지만 가장 낮은 우선순위.

가능 경로:

```text
팀원/교수/코치/지인
  ↓
건설사
  ↓
현장소장/안전관리자
  ↓
짧은 검증
```

요청 방식도:

```text
X 시스템 설치/운영을 허용해 달라
O 작업 없는 지정 구역에서 팀원만 대상으로 1~2시간 카메라 검증
```

가 현실적이다.

실제 작업자를 촬영하면 개인정보/노무/보안 문제가 급격히 커진다.

---

# 49. 시연환경 3단계 구조

최종 추천:

## Level 1 — Development / Live Demo

```text
SSAFY Project Room
```

목적:
- 반복 개발
- live demo
- deterministic scenario
- calibration
- latency
- event flow

## Level 2 — External Validation

```text
안전체험교육장
또는
산업형 공간
```

목적:
- industrial visual domain
- generalization
- occlusion
- real background
- false positive stress test

## Level 3 — Real-world Evidence

```text
AI-Hub
SARD
ConstrID
공개 건설영상
가능하면 실제 현장 단기 검증
```

목적:
- real construction domain
- generalization report
- replay
- evaluation

---

# 50. 최종 발표 구성

발표 논리:

> 실제 공사현장에서는 개인정보와 안전 문제로 위험 사고를 직접 재현할 수 없으므로, 동일한 AI/Spatial Pipeline을 세 단계 환경에서 검증했다.

## 1) SSAFY LIVE

```text
CAM01
 ↓
Person
 ↓
Track
 ↓
PPE
 ↓
World XY
 ↓
Hazard Zone
 ↓
Risk Queue
 ↓
Alert
```

## 2) External Domain Test

- 다른 background
- 산업 구조물
- occlusion
- 조명 차이

성능:
- Recall
- FP
- Tracking
- Position Error

## 3) Real Construction Replay

실제 공개 데이터:
- 동일 inference
- event extraction
- 2D/2.5D replay

## 4) Unreal Extension

가능할 경우:
- external validation space를 3DGS capture
- Unreal Reality Snapshot
- 동일 Event를 3D에 overlay
- T0/T1/T2 timeline

---

# 51. 최종 전체 시스템 구조

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OFFLINE KNOWLEDGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AI-Hub
SARD
PPE Public Data
Tracking Data
Crack Data
Fire Data
Synthetic Rare Events
        │
        ▼
Construction-domain Models


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REAL SITE / DEMO SITE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Fixed CCTV ──────────────┐
                         │
Inspection Smartphone ───┤
                         │
Equipment Telematics ────┤ optional
                         │
Thermal ─────────────────┤ optional
                         │
Drawing / BIM ───────────┘
             │
             ▼
      Site-specific Data
             │
             ▼
       Active Learning


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INFERENCE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Detection
   +
Tracking
   +
Pose / Temporal
   +
Spatial Projection
   +
Sensor Fusion
        │
        ▼
Observation
        │
        ▼
Spatial / Temporal Engine
        │
        ▼
Risk Queue
        │
        ▼
Decision
        │
        ▼
Event


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRESENTATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2D / 2.5D Monitoring
        │
        ├ CCTV Evidence
        ├ Position
        ├ Trajectory
        ├ Zone
        ├ Timeline
        └ Report
               │
               ▼
     Optional Extension
      Unreal + 3DGS
```

---

# 52. 프로젝트의 핵심 기술축 재정의

이 세션을 거치며 프로젝트의 중심은 다음으로 정리되었다.

단순:

```text
YOLO 돌리기
```

가 아니다.

정확히는:

```text
Reality / Sensor
     ↓
Detection
     ↓
Tracking
     ↓
Spatial Mapping
     ↓
Temporal Context
     ↓
Risk Reasoning
     ↓
Event
     ↓
2D/2.5D / Unreal
```

사용자 담당 영역과 연결하면:

```text
AI Model
    ↓
AI Pipeline
    ↓
Spatial AI
    ↓
Coordinate Transform
    ↓
Event Contract
    ↓
Digital Twin
```

즉 AI 모델과 Backend 사이의 **공간·시간 의미 변환 계층**이 핵심 역할이다.

---

# 53. 앞으로의 구현 우선순위

## P0 — 먼저 해야 할 것

1. CAM01 1대 기준
2. 도면/모의평면 정의
3. 기준점 4개+
4. Homography
5. Person/PPE detection
6. Tracking
7. World XY
8. Zone
9. Risk Queue
10. Event Schema
11. 2D/2.5D Dashboard

## P1 — 데이터

1. AI-Hub PPE
2. AI-Hub 고소작업
3. AI-Hub 위험상태
4. SARD
5. 자체 CAM01
6. Hard-negative
7. Site Fine-tuning

## P2 — 외부 검증

1. SSAFY live
2. 안전체험시설 또는 산업형 공간
3. 공개 실제 건설영상 replay

## P3 — 3D 확장

1. 스마트폰 reality capture
2. COLMAP
3. gsplat/Postshot
4. World Registration
5. Unreal
6. Event Overlay
7. Snapshot Timeline

## P4 — 조건부 확장

- CCTV 2대째
- LiDAR/RGB-D
- thermal
- telematics
- crack pipeline
- true 4DGS

---

# 54. 명시적으로 버린 방향

본 세션에서 다음 방향은 MVP 기준으로 버리거나 후순위로 내렸다.

### 버림 / 후순위

- 3DGS를 프로젝트 코어로 삼기
- 실제 공사현장에 이동 카메라 상시 운영
- 시작부터 CCTV 3~4대
- 시작부터 LiDAR/RGB-D
- 360 camera 필수
- 단일 CCTV에서 정확한 3D XYZ 전체 추정
- 균열을 멀리 있는 CCTV로 탐지
- 모든 위험을 하나의 end-to-end 모델로 판단
- 모든 센서 문제를 vision으로 해결
- 실제 사고 시나리오 직접 재현
- 실제 공사현장을 최종 시연의 필수조건으로 설정
- 시작부터 true 4DGS

---

# 55. 현재 남은 주요 설계 과제

다음 문서/설계가 이어져야 한다.

## A. `SENTIA_DATA_PIPELINE_v1`

포함할 것:

- 공개 dataset 다운로드 목록
- 이용조건/라이선스
- 채택 class
- unified taxonomy
- dataset conversion
- train/val/test 정책
- own-site capture
- hard-negative mining
- active learning
- model별 training input

## B. `SENTIA_SPATIAL_PIPELINE_v1`

포함할 것:

- camera calibration
- homography
- coordinate conventions
- floor/zone
- worker bottom-center
- position smoothing
- trajectory
- speed
- dwell
- equipment proximity
- spatial confidence

## C. `SENTIA_EVENT_CONTRACT_v1`

포함할 것:

- Observation
- State
- Risk Queue
- Event
- evidence
- severity
- confidence
- retry/cooldown
- Backend API
- Web/Unreal consumer contract

## D. `SENTIA_DEMO_PROTOCOL_v1`

포함할 것:

- SSAFY Micro Construction Zone
- camera position/FOV
- zone layout
- PPE scenarios
- safety constraints
- repeatable demo sequence
- external validation protocol
- replay protocol
- evaluation metrics

## E. `SENTIA_3DGS_EXTENSION_v1`

포함할 것:

- capture protocol
- frame extraction
- COLMAP
- gsplat/Postshot
- world registration
- Unreal plugin
- proxy mesh/BIM
- snapshot timeline
- event overlay

---

# 56. 참고 자료 목록

## 3DGS / Reconstruction / Unreal

- 3D Gaussian Splatting  
  https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/

- COLMAP  
  https://colmap.github.io/

- gsplat  
  https://docs.gsplat.studio/

- Postshot  
  https://www.jawset.com/docs/d/Postshot%2BUser%2BGuide

- Nerfstudio custom dataset  
  https://docs.nerf.studio/quickstart/custom_dataset.html

- XGRIDS LCC Unreal Plugin  
  https://github.com/xgrids/LCC-3DGS-Unreal-Plugin

- DazaiStudio SplatRenderer  
  https://github.com/DazaiStudio/SplatRenderer-UEPlugin

- YAGS  
  https://github.com/yandex/yags

## Construction Reality / Digital Twin

- OpenSpace Capture  
  https://www.openspace.ai/products/capture/

- OpenSpace Best Practices  
  https://support.openspace.ai/hc/en-us/articles/41951421694099-OpenSpace-Best-Practices

- Digital Twin Consortium Reality Capture  
  https://www.digitaltwinconsortium.org/wp-content/uploads/sites/3/2022/06/Reality-Capture-A-Digital-Twin-Foundation.pdf

## Construction AI / Open Data

- SARD  
  https://zenodo.org/records/19926448

- ConstrID  
  https://github.com/Drlijiaqi/ConstrID-Dataset

- Worker-Safety-Twin  
  https://github.com/almosenja/Worker-Safety-Twin

- PPE-Detection-for-Construction-Site-Safety  
  https://github.com/NamHoKi/PPE-Detection-for-Construction-Site-Safety

- Tracking-and-Material-Counting  
  https://github.com/smartconstructiongroup/Tracking-and-Material-Counting

- Atharion PPE  
  https://github.com/atharion-team/ppe-dataset-v1

- D-Fire  
  https://github.com/gaia-solutions-on-demand/DFireDataset

- VTT-ConIoT  
  https://zenodo.org/records/4683703

- SynthSite  
  https://huggingface.co/datasets/govtech/SynthSite

## AI-Hub

- 공사현장 안전장비 인식 데이터
- 고소작업 현장 실시간 영상 데이터
- 건설 현장 위험 상태 판단 데이터
- 건설 현장 장비 모니터링 및 생산성 측정 데이터
- SOC 시설물 균열 데이터
- 무인 플랜트 안전 감시 데이터
- 이상행동 CCTV 데이터
- 지하철 CCTV 데이터

AI-Hub:
- https://aihub.or.kr/

## Crack

- METU Concrete Crack Images  
  https://data.mendeley.com/datasets/5y9wdsg2zt/2

- Crack Segmentation Dataset  
  https://data.mendeley.com/datasets/jwsn7tfbrp/1

- DeepCrack  
  https://github.com/yhlleo/DeepCrack

---

# 57. 최종 Canonical Statement

SENTIA의 현실 연결부는 다음과 같이 정의한다.

> **공개 건설 데이터로 AI의 일반 능력을 확보하고, 고정 CCTV 1대의 실제 사용 환경 데이터로 현장 적응을 수행한다. CCTV에서는 Detection을 넘어서 Tracking과 좌표 변환을 통해 작업자·장비의 공간 상태를 얻고, 이를 Zone·시간·관계 기반 Risk Engine으로 가공한다. 균열·열·장비 위치처럼 더 적합한 센서가 존재하는 문제는 CCTV에 억지로 통합하지 않는다. 실시간 제품의 중심은 2D/2.5D 관제이며, 3DGS는 관리자의 주기적 스마트폰 촬영을 이용해 생성하는 Reality Snapshot으로 Unreal 확장팩에 결합한다. 개발 및 최종 라이브 시연은 SSAFY 내부의 안전한 Micro Construction Zone에서 수행하고, 외부 산업안전체험시설 또는 산업형 공간에서 도메인 검증하며, 실제 건설현장 공개 데이터로 Generalization Evidence를 확보한다.**
