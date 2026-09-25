# SENTIA 범위와 계획

> 이 문서는 01~03에서 확정한 제품 방향을 실제 자율 프로젝트 기간 안에서 어떻게 구현할지 관리한다.
> 제품의 뼈대를 임의로 다시 정의하지 않고, **구체 사건·데이터·단말·기술 깊이**를 팀 합의로 결정한다.

## 문서 역할

01~03과의 관계는 다음과 같다.

~~~text
01
왜 만드는가
세 Pain Point와 제품 가치

        ↓

02
누가 무엇을 책임지는가

        ↓

03
시스템이 어떤 정보 흐름으로 동작하는가

        ↓

04
이번 일정에서
무엇을 어느 깊이까지 실제 구현할 것인가
~~~

이 문서는 API 명세, DB 스키마, 개인별 Sprint Backlog, 모델 세부 설계서를 대체하지 않는다.

## 계획 기준

소스 우선순위에 따라 다음 제품 뼈대는 이미 방향이 정해진 것으로 취급한다.

1. **오탐·맥락 부족 대응**
   - Event Candidate
   - Event Queue
   - Context / Spatial / Temporal
   - 필요 시 VLM
   - Priority / Risk State
   - 관리자 1차 승인·기각
   - False Alarm 분기

2. **공간 정보 단절 대응**
   - Site / Floor / Zone
   - Spatial State
   - 도면 중심 2D/2.5D 관제

3. **조치·기록 단절 대응**
   - 승인된 사건의 Action
   - 현장 전달
   - ACK / Verify / Evidence
   - History
   - AI Report Draft
   - 안전관리자 검토 후 업무 문서 활용

이 세 축을 통째로 없애는 것은 단순 Scope 조정이 아니라 제품 정의 변경이다.
필요하면 팀이 명시적으로 다시 합의해야 한다.

반대로 다음은 아직 팀이 결정할 수 있다.

- 어떤 Event를 실제 시연할지
- 어떤 Source를 사용할지
- 어떤 단말로 현장 전달을 구현할지
- 어떤 보고서 종류를 생성할지
- VLM을 어느 Event에 사용할지
- 2.5D를 어디까지 구현할지
- 3DGS·BIM·Unreal을 실제로 넣을지

## 일정 전제

### 공식 일정

| 날짜 | 일정 | 계획상 의미 |
|---|---|---|
| 9/28~10/2 | 공통 프로젝트 마무리 + 자율 프로젝트 팀 빌딩 | 사전 준비 |
| 10/5 | 개천절 대체공휴일 | 정상 개발일 제외 |
| 10/6 | 실질 정식 작업 시작 | 프로젝트 착수 |
| 10/8 | 필드트립 | 정상 개발일 제외 |
| 10/9 | 한글날 | 공휴일 |
| 10/16 | 중간평가 | 첫 통합 상태 확인 |
| 11/13 | 권장 Feature Freeze | 신규 기능 개발 종료 |
| 11/16~11/18 | 최종 안정화 | 검증·리허설·발표 준비 |
| 11/18 | 최종평가 | 개발 Hard Stop |
| 11/23~11/24 | 본선 발표회 | 평가 이후 발표 |
| 12/1 | 전시발표회 | 결과물 전시 |

본선 발표회와 전시발표회는 Core 개발 기간으로 계산하지 않는다.

### 첫 주

10/5 대체공휴일과 10/9 한글날, 10/8 필드트립을 고려하면 정식 프로젝트 첫 주의 정상 작업일은 10/6과 10/7뿐이다.

따라서 첫 주를 일반적인 5일 Sprint처럼 계산하지 않는다.

### 실질 기간

10/5~11/18의 평일은 약 33일이다.

다음 일정을 제외하면 실제 정상 작업 가능일은 약 30일 수준이다.

- 10/5 대체공휴일
- 10/8 필드트립
- 10/9 한글날

11/13을 Feature Freeze로 두면 신규 기능 개발에 사용할 수 있는 정상 작업일은 약 27일 수준이다.

따라서 과거의 단순 “8주 프로젝트” 전제는 사용하지 않는다.

## 계획 원칙

### 1. 업무 우선

~~~text
안전관리 업무
→ Pain Point
→ SENTIA 개입
→ 필요한 현실 정보
→ 기능
→ 기술
~~~

기술부터 선택하고 업무를 맞추지 않는다.

### 2. 세 Pain Point 추적

모든 Scope 결정은 다음 질문을 통과해야 한다.

~~~text
오탐·맥락 문제를 어떻게 줄이는가?
공간 정보 단절을 어떻게 해결하는가?
조치·기록 단절을 어떻게 닫는가?
~~~

특정 기능이 화려해도 세 제품 축과 관계가 약하면 우선순위를 낮출 수 있다.

### 3. Early E2E

각 파트의 독립 완성보다 작은 End-to-End 흐름을 먼저 만든다.

~~~text
Reality Input
→ Detection / Anomaly
→ Event Candidate
→ Event Queue
→ Context / Spatial
→ 관리자 Review
→ 승인 / 기각
→ Action
→ Verify / History
→ Report Draft
~~~

중간 단계는 초기 Vertical Slice에서 Stub 또는 최소 구현일 수 있다.
그러나 Core 단계에서는 선택한 Main Story가 실제로 끝까지 이어져야 한다.

### 4. Human-in-the-loop

AI가 자동으로 최종 위험 판단이나 공식 보고서를 확정하지 않는다.

~~~text
후보
→ 관리자 승인 / 기각

Report Draft
→ 안전관리자 검토 / 수정
~~~

### 5. Core before Expansion

3DGS, Unreal, 추가 Sensor, Multi-camera 같은 확장은 Core가 안정된 뒤에만 진행한다.

### 6. 마지막 주는 개발 주간이 아니다

11/16 이후 신규 기능 추가를 원칙적으로 금지한다.

## Pain Point Coverage

| Pain Point | 제품 Backbone | Scope에서 정할 것 |
|---|---|---|
| 오탐·맥락 부족 | Event Queue + Context/VLM/Spatial + 관리자 승인/기각 | Event 종류, VLM 범위, Risk 기준 |
| 공간 정보 단절 | Site/Floor/Zone + Spatial State + 공간 관제 | Calibration 깊이, 2D/2.5D 범위 |
| 조치·기록 단절 | Action Handoff + ACK/Verify/Evidence + AI Report Draft | Endpoint 형태, 보고서 종류·형식 |

Scope 회의에서 기능을 줄이더라도 이 표를 기준으로 **어느 Pain Point의 해결력이 약해지는지 명시적으로 확인한다.**

## 팀 합의 항목

### 제품 Story

| 항목 | 상태 | 결정 기준 |
|---|---|---|
| Main Demo Story의 구체 사건 | 팀 합의 필요 | 업무 가치, 시연성, 데이터 |
| 핵심 업무 범위의 깊이 | 팀 합의 필요 | 실제 안전관리 업무, 일정 |
| 핵심 관측·사건 | 팀 합의 필요 | 업무 가치, Data/Testbed |
| Must / Should | 팀 합의 필요 | Main Story와 세 Pain Point |

Main Demo Story의 **구조적 뼈대**는 이미 정해져 있다.

~~~text
현실 관측
→ Event Candidate
→ 맥락·공간 판단
→ 관리자 승인/기각
→ 승인 시 현장 조치
→ 검증·Evidence
→ AI Report Draft
~~~

팀이 정할 것은 이 흐름을 어떤 사건과 어느 구현 깊이로 보여 줄지다.

### 오탐 판단

| 항목 | 상태 |
|---|---|
| 승인/기각 Human Gate | 방향 확정 |
| False Alarm 기록 | 방향 확정 |
| Priority/Risk State 세부 계산 | 팀 합의·검증 필요 |
| VLM 적용 Event | 팀 합의·검증 필요 |
| False Alarm을 Hard Negative로 활용하는 범위 | 팀 합의 필요 |
| 자동 재학습 | 미확정, 기본 전제 아님 |

### 현장 조치 전달

Notion의 Target UX에는 안전관리자와 작업자의 스마트워치가 포함되어 있다.

따라서 **현장 Action Handoff 자체는 제품 방향에서 제외하지 않는다.**

팀이 결정할 것은 이번 프로젝트의 구현 형태다.

| 선택 후보 | 상태 |
|---|---|
| Smartwatch Native | 팀 합의 필요 |
| Mobile Web/App | 팀 합의 필요 |
| Web Endpoint | 팀 합의 필요 |
| Testbed용 Mock Endpoint | 팀 합의 필요 |

최소한 승인된 Action이 관리자 화면에만 남고 현장에 전달되지 않는 구조는 피한다.

### AI 보고서

History/Evidence 기반 AI Report Draft는 제품 방향에 포함한다.

팀이 결정할 항목:

- 일일 안전일지
- 조치보고
- 아차사고 기록
- 위험성평가 Feedback
- 기타 내부 보고서

또한 다음을 정한다.

- 어떤 모델을 사용할지
- Template 기반인지 자유 생성 기반인지
- Evidence Reference를 어떻게 포함할지
- 어떤 형식으로 출력할지
- 이번 프로젝트에서 몇 종류를 구현할지

최종 보고서는 안전관리자가 검토·수정하는 구조를 유지한다.

### 관측 범위

다음은 모두 후보이며 우선순위를 팀이 결정한다.

| 후보 | 계열 | 상태 |
|---|---|---|
| PPE 미착용 | 동적 관측 | TBD |
| 위험구역 진입 | 동적 관측 | TBD |
| 작업자·중장비 접근 | 동적 관측 | TBD |
| 작업자 쓰러짐 | 동적 관측 | TBD |
| 추락·낙하·충돌·전도 | 동적 관측 | TBD |
| 균열·구조물 결함 | Inspection | TBD |
| 화재·연기·과열 | 환경 관측 | TBD |
| 가스·누출 | 환경 관측 | TBD |
| 공정·현장 변화 | Reality / Change | TBD |

관측 대상이 늘어나는 만큼 별도 Model/Pipeline 부담이 증가하므로 “많이 탐지한다” 자체를 목표로 삼지 않는다.

### 공간·확장

| 항목 | 상태 |
|---|---|
| Site/Floor/Zone 기반 공간 연결 | 방향 확정 |
| 2D 도면 관제 | 방향 확정 |
| 2.5D 구현 깊이 | 팀 합의 필요 |
| 3DGS | 조건부 |
| BIM 고도화 | 조건부 |
| Unreal | 조건부 |
| Thermal / Telematics / Wearable | 조건부 또는 관측 선택에 따라 결정 |

## 결정 기록

팀 합의는 다음 형식으로 남긴다.

~~~text
항목
결정
근거
대안
결정일
재검토 조건
영향받는 Pain Point
영향받는 역할
~~~

Scope 변경 시 “무엇을 뺐는가”뿐 아니라 **어떤 Pain Point와 역할에 영향을 주는지** 같이 기록한다.

## 범위 분류

| 분류 | 의미 |
|---|---|
| Must | Main Story와 제품 Backbone을 성립시키는 데 필수 |
| Should | 가치가 높지만 Core를 깨지 않는 범위에서 구현 |
| Conditional | Gate 통과 이후에만 착수 |
| Out | 이번 일정에는 구현하지 않음 |

Out은 아이디어 폐기가 아니다.

## 데이터 계획

핵심 Event가 정해진 뒤 다음을 분리해 관리한다.

| 목적 | 의미 |
|---|---|
| Training | 모델 학습·Fine-tuning |
| Validation / Test | 실제 성능 검증 |
| Demo | 평가 환경에서 재현할 시연 데이터 |

권장 기록 구조:

| Event | Training | Validation/Test | Demo | 자체 촬영 | 확보 상태 | Risk |
|---|---|---|---|---|---|---|
| TBD | TBD | TBD | TBD | TBD | TBD | TBD |

Discovery에서 조사된 공개 Dataset은 후보 자료다.
Event 결정 전에 특정 Dataset을 프로젝트 표준으로 고정하지 않는다.

## 시연 환경

현재 가능한 조합:

- Recorded / Public Replay
- Controlled Testbed
- 실제 현장에서 확보한 자료
- 실제 건설 현장
- 위 방식의 혼합

실제 건설 현장 접근을 프로젝트 성공의 필수조건으로 두지 않는다.

위험한 사고를 Testbed에서 실제로 재현하지 않는다.
희귀·고위험 사건은 공개 데이터, Replay, 안전한 Proxy, Synthetic 등을 사용할 수 있다.

## 단계 계획

~~~mermaid
gantt
    title SENTIA 자율 프로젝트 일정
    dateFormat YYYY-MM-DD
    axisFormat %m/%d
    excludes weekends, 2026-10-05, 2026-10-08, 2026-10-09

    section 준비
    사전 준비          :prep, 2026-09-28, 5d

    section 개발
    1차 범위 결정      :scope, 2026-10-06, 2d
    Scope Gate         :milestone, sg, 2026-10-12, 0d
    수직 완성          :slice, 2026-10-12, 5d
    핵심 제품          :core, 2026-10-19, 10d
    고도화             :adv, 2026-11-02, 5d
    조건부 확장        :ext, 2026-11-09, 5d

    section 마무리
    Feature Freeze     :milestone, ff, 2026-11-13, 0d
    최종 안정화        :stab, 2026-11-16, 3d
    최종평가           :milestone, final, 2026-11-18, 0d
~~~

### 사전 준비 — 9/28~10/2

공통 프로젝트 마무리와 팀 빌딩 기간이다.

진행 가능:

- 01~04 공유
- Notion·Discovery 공유
- 안전관리 업무·Pain Point 확인
- Event 후보와 Dataset 조사
- Testbed 가능성 확인
- 기술 Spike 목록 작성
- 개발환경 준비

정식 Sprint로 계산하지 않는다.

### 1차 범위 결정 — 10/6~10/7

정상 작업일이 이틀뿐이다.

목표:

- Main Story 후보 축소
- 핵심 Event 후보 축소
- 세 Pain Point Coverage 확인
- Data Source 후보
- 기술 Spike 착수
- 역할별 부담 확인

모든 Scope를 이틀 안에 영구 고정하지 않는다.

### Scope Gate — 10/12 초기

10/8 필드트립 결과를 포함해 첫 Vertical Slice에 필요한 범위를 잠정 확정한다.

최소 확인:

- 어떤 Event로 시연하는가
- 어떤 Source를 쓰는가
- Event Candidate를 만들 수 있는가
- Spatial Context를 제공할 수 있는가
- 승인/기각을 보여 줄 수 있는가
- 승인 후 Action을 어디로 전달할 것인가
- Verify/Evidence를 어떻게 돌려받을 것인가
- AI Report Draft를 어떤 최소 형태로 연결할 것인가
- 역할별 일정이 감당 가능한가

Scope Gate는 영구 Freeze가 아니다.
Data/기술 실패 시 범위를 줄일 수 있다.

### 수직 완성 — 10/12~10/16

중간평가까지 **작지만 실제로 연결된 SENTIA**를 만든다.

권장 Vertical Slice:

~~~text
Reality Input
→ Detection / Anomaly
→ Event Candidate
→ 공간 위치
→ 관리자 Review
→ 승인 또는 기각
~~~

가능하면 승인 경로에 최소 Action State까지 연결한다.

Report Generation과 실제 현장 Endpoint는 이 단계에서 Mock/Stub일 수 있다.
단, Core 단계에서 반드시 실제 흐름으로 이어질 수 있는 Contract는 잡아둔다.

완료 조건:

- 한 사건이 E2E로 재현된다.
- 2D 공간에서 위치를 확인할 수 있다.
- 관리자가 승인·기각할 수 있다.
- 기각은 False Alarm으로 분리된다.
- 승인된 후보가 다음 Action 단계로 넘어갈 수 있다.

### 핵심 제품 — 10/19~10/30

이 단계 종료 시 조건부 확장이 하나도 없어도 SENTIA의 세 Pain Point가 모두 설명돼야 한다.

Core Exit Criteria:

#### 오탐·맥락

- Event Candidate와 즉시 Alert가 구분된다.
- Event Queue가 존재한다.
- 선택한 Context가 반영된다.
- 관리자 승인·기각이 동작한다.
- False Alarm이 별도 기록된다.

#### 공간

- 선택한 Reality Source와 도면/Zone이 연결된다.
- Spatial State를 관제에서 확인할 수 있다.
- 위치 판단이 Screen Pixel에 종속되지 않는다.

#### 조치·기록

- 승인된 SafetyEvent가 Action으로 이어진다.
- 선택한 현장 Endpoint 또는 동등한 Test Endpoint에 전달된다.
- ACK 또는 완료 상태를 받을 수 있다.
- Verification/Evidence가 History로 남는다.
- 구조화 History를 바탕으로 AI Report Draft를 생성한다.
- 안전관리자가 Draft를 검토할 수 있다.

이 시점이 실제 본제품 완성선이다.

### 고도화 — 11/2~11/6

Core를 더 설득력 있게 만든다.

후보:

- 오탐 감소와 Context 강화
- VLM
- Priority/Risk State 고도화
- Hard Negative 관리
- AI Report 품질 개선
- Report Template 추가
- Action/Notification UX 개선
- 2.5D
- Before/After
- 성능·Monitoring
- Evidence UX

무엇을 선택할지는 Core 상태와 팀 합의를 따른다.

### 조건부 확장 — 11/9~11/13

특정 기술에 미리 예약하지 않는다.

Core와 Demo가 안정적일 때만 다음을 고려한다.

- 3DGS
- Unreal
- BIM 고도화
- Multi-camera / ReID
- Thermal / Sensor
- 추가 Event
- 고급 AI 기능

Core가 흔들리면 즉시 NO-GO하고 다음에 사용한다.

- Bugfix
- 오탐 개선
- Action Handoff 안정화
- AI Report 품질
- UX
- 성능
- Demo Fallback

11/13 종료 시 Feature Freeze한다.

### 최종 안정화 — 11/16~11/18

신규 기능 추가를 원칙적으로 금지한다.

- 통합 테스트
- Demo Data 고정
- 서버·GPU·Network 검증
- Watch/Mobile/Web Endpoint Fallback
- AI Report 결과 재현성 확인
- VLM/AI 실패 Fallback
- 발표 Scenario
- 리허설
- 시연 영상
- 문서
- 최종 디자인
- 장애 대응

## 진행 Gate

### Pain Point Coverage Gate

Scope 또는 기능을 줄일 때 다음 세 질문을 확인한다.

1. 관리자 승인·기각까지의 오탐 제어 Story가 남아 있는가?
2. 사건을 현장 공간에서 설명할 수 있는가?
3. 승인된 사건이 현장 Action과 AI Report까지 이어지는가?

하나가 빠질 경우 팀이 명시적으로 Product Scope 변경으로 기록한다.

### Vertical Slice Gate

중간평가 전후 확인:

~~~text
Input
→ Processing
→ Candidate
→ Spatial
→ Human Review
~~~

연결이 실패하면 새 기능보다 통합을 우선한다.

### Core Gate — 10월 말

확인:

- Main Story 완주
- 세 Pain Point 모두 최소 한 번 제품 기능으로 증명
- 핵심 Data 확보
- AI/Reality Processing 안정
- Spatial Mapping 안정
- 승인·기각 동작
- Action Handoff 동작
- Verify/Evidence/History 동작
- AI Report Draft 동작
- Demo 가능

### Expansion Gate — 11월 초

다음 조건이 모두 충족되어야 한다.

- Core Gate 통과
- Blocking Issue 없음
- 파트별 Must 완료
- Demo Flow 확보
- 확장을 끝낼 시간이 충분함

아니면 NO-GO.

NO-GO는 실패가 아니라 Core 완성도를 지키는 결정이다.

## Workload 검토

| 범위 추가 | 부담 증가 |
|---|---|
| Event/Model 추가 | AI-1 |
| Tracking/Spatial/VLM/Report AI | AI-2 |
| Context/Risk/Review 정책 | BE-1 |
| Action/Evidence/Report Workflow | BE-2 |
| Realtime/Notification/Endpoint | BE-3 |
| 공간/Review/Action/Report UI | FE-1 |

부담이 한 역할에 몰리면 역할 경계를 흐리는 대신 Scope를 줄이는 것을 우선한다.

## 주요 Risk

| Risk | 대응 |
|---|---|
| Event 종류 과다 | Main Story 기준으로 축소 |
| Data 부족 | Scope Gate 전 검증 |
| 실제 현장 미확보 | Replay/Testbed Fallback |
| AI 오탐 | Candidate Gate + Context + Human Review |
| Spatial 오차 | Calibration 실측 |
| VLM 지연 | Realtime과 Work Plane 분리 |
| 현장 전달 실패 | Endpoint Fallback |
| AI Report 환각·부정확 | 구조화 Context + Evidence + Human Review |
| 파트별 병렬 개발 후 통합 실패 | Early E2E |
| 확장 일정 잠식 | Expansion Gate |
| 특정 역할 과부하 | Scope 축소 |
| 마지막 주 기능 추가 | 11/13 Freeze |

## 현재 상태

| 상태 | 내용 |
|---|---|
| 제품 방향 확정 | 세 Pain Point를 모두 제품 구조로 해결<br>Event Queue와 관리자 승인·기각<br>Site/Floor/Zone 기반 공간 관제<br>승인 후 현장 Action Handoff<br>ACK/Verify/Evidence/History<br>History 기반 AI Report Draft + Human Review |
| 팀 합의 필요 | Main Demo Event<br>관측 Source<br>Must/Should 깊이<br>현장 Endpoint 구현 형태<br>AI Report 종류·형식<br>VLM 적용 범위<br>2.5D 깊이 |
| 검증 필요 | Dataset<br>모델·오탐 성능<br>Tracking/Calibration<br>Latency/Queue<br>Endpoint 재현성<br>AI Report 품질<br>GPU·Testbed |
| 조건부 | 3DGS<br>BIM 고도화<br>Unreal<br>Multi-camera/ReID<br>Thermal/Telematics/Wearable<br>기타 확장 |

팀 합의가 끝날 때마다 이 문서를 갱신한다.
