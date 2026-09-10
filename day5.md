좋아. 그럼 **5명 × 5일** 기준으로, 실제로 첫날부터 뭘 해야 하는지까지 내려서 계획을 짜보자. 핵심은 “5일 동안 기능을 많이 넣는 것”이 아니라 **Day 1에 SDK baseline을 확보하고, Day 2~3에 팀이 병렬 작업할 수 있는 인터페이스를 고정하는 것**이야.

자료상 APACHE6 SDK에는 `applications` 아래 샘플 애플리케이션과 공통 라이브러리가 있고, Wayland NPU 샘플은 `V4L2 → render() → cnn_task → NPU → pp_task → Wayland/Overlay Mixer` 흐름이다. SDK 문서에도 application만 별도 `build.sh`로 빌드해서 `output/user_data`로 복사하고, NFS/SD/eMMC에서 실행하는 흐름이 제공된다.  

다만 첨부 문서에는 **Trichimera 샘플의 정확한 소스 파일명/함수명 전체가 공개되어 있지 않다.** 그래서 아래에서 `cnn_task`, `pp_task`, `render()`처럼 문서에서 확인되는 함수명은 그대로 쓰고, 나머지 파일명은 실제 SDK 소스에서 첫날 확인해서 매핑해야 한다.

---

# 전체 목표

금요일 최종 실행 프로그램 하나를 만드는 걸 목표로 한다.

```text
Camera
  ↓
V4L2
  ↓
Trichimera @ NPU
  ↓
Detection + Freespace + Lane
  ↓
Risk Engine
  ↓
SAFE / WARNING / DANGER
  ↓
Wayland Overlay
```

그리고 소프트웨어 구조는 최종적으로 이렇게 나누자.

```text
src/
 ├─ ai/
 │   └─ ai_adapter.*
 │
 ├─ risk/
 │   ├─ geometry.*
 │   └─ risk_engine.*
 │
 ├─ system/
 │   ├─ shared_state.*
 │   ├─ perf_monitor.*
 │   └─ signal_handler.*
 │
 ├─ ui/
 │   └─ adas_overlay.*
 │
 └─ main / 기존 SDK application
```

실제 SDK 디렉터리를 마음대로 바꾸라는 뜻은 아니고, **논리적으로 이 단위로 담당 영역을 분리하자**는 의미다.

---

# 팀원 5명 역할

처음부터 아래처럼 고정하는 게 좋다.

| 팀원 | 주 담당               | 최종 산출물                       |
| -- | ------------------ | ---------------------------- |
| A  | AI/NPU             | `AIResult` 구조체까지 결과 전달       |
| B  | Geometry           | freespace/lane/point 판정      |
| C  | Risk               | SAFE/WARNING/DANGER 로직       |
| D  | UI                 | bbox/mask/lane/risk overlay  |
| E  | System/Integration | thread/mutex/signal/build/통합 |

여기서 E는 “남는 일 하는 사람”이 아니다. 오히려 **전체 프로그램 구조를 관리하는 사람**으로 잡아야 한다.

---

# 시작하기 전에 Git부터 정리

5일밖에 없으니 Git도 단순하게 한다.

```text
main
 ├─ feat/ai
 ├─ feat/geometry
 ├─ feat/risk
 ├─ feat/ui
 └─ feat/system
```

원칙은 세 개만.

1. `main`은 항상 실행 가능한 상태
2. 기능 하나 끝날 때마다 PR/merge
3. 여러 사람이 SDK 원본 파일 하나를 동시에 수정하지 않기

특히 기존 sample의 `main`이나 `render()`를 A, D, E가 동시에 수정하기 시작하면 충돌 때문에 시간이 날아간다.

그래서 **원본 application 수정 권한은 통합 담당 E가 중심**, 나머지는 가능하면 새 `.cpp/.h` 또는 `.c/.h` 모듈을 만든다.

---

# DAY 1 — “무조건 기존 AI 데모 실행”

Day 1 목표는 개발이 아니다.

**카메라 → Trichimera → 화면 출력이 정상 동작한다는 걸 확보하는 날**이다.

SDK에는 Wayland NPU Application 자체가 카메라 입력을 받아 NPU에서 Object Detection + Freespace + Lane Detection을 수행하고 결과를 Wayland로 출력하는 샘플로 설명돼 있다. 

## 09:00–10:00 전체 팀

### ① 개발환경 확인

호스트:

```bash
pwd
git status
git branch
```

SDK 상위 구조 확인:

```bash
ls
```

문서 기준으로 대략:

```text
linux-sdk-apache6/
 ├─ applications
 ├─ arm-trusted-firmware
 ├─ buildroot
 ├─ linux-kernel
 ├─ modules
 ├─ tools
 ├─ u-boot
 └─ ...
```

이 구조는 Quick Guide에도 명시돼 있다. 

---

### ② applications 구조 확인

```bash
cd applications
find . -maxdepth 2 -type f | less
```

특히 찾아야 할 것:

```bash
find . -iname '*npu*'
find . -iname '*wayland*'
find . -iname '*trichimera*'
find . -iname '*.aiwbin'
find . -iname '*.ini'
```

소스에서도:

```bash
grep -R "cnn_task" -n .
grep -R "pp_task" -n .
grep -R "render(" -n .
```

**이 세 grep 결과는 팀 채팅에 바로 공유한다.**

왜냐하면 문서에서 확인되는 핵심 흐름이 바로 이 부분이기 때문이다. 

---

# 10:00–12:00

## A — AI 담당

찾는다.

```text
Trichimera model load 위치
NPU init 위치
cnn_task
inference 호출부
post-processing callback
Detection 결과 구조
Freespace 결과 구조
Lane 결과 구조
```

코드를 수정하지 말고 우선 메모한다.

예:

```text
AI init:
xxx.cpp:142

cnn_task:
xxx.cpp:381

post process:
xxx.cpp:511
```

그리고 **출력 타입을 종이에 그린다.**

목표:

```cpp
struct AIResult {
    ObjectInfo objects[...];
    int object_count;

    FreespaceInfo freespace;
    LaneInfo lane;
};
```

아직 구현 안 해도 된다.

---

## B — Geometry 담당

A와 같이 post-processing 코드를 본다.

확인 대상:

```text
Segmentation 결과가
- 픽셀 mask인가?
- polygon인가?
- overlay용 buffer인가?

Lane 결과가
- point 배열인가?
- polynomial coefficient인가?
- lane index인가?
```

이걸 모르면 geometry 모듈을 설계할 수 없다.

---

## C — Risk 담당

이때는 SDK 건드리지 않는다.

PC에서 독립적으로 risk API부터 설계.

```cpp
enum RiskLevel {
    RISK_SAFE,
    RISK_WARNING,
    RISK_DANGER
};
```

```cpp
RiskLevel evaluateRisk(...);
```

그리고 종이에 판단 흐름 작성.

```text
Object
 ↓
bottom center
 ↓
Freespace?
 NO → SAFE
 YES
 ↓
Ego lane?
 NO → SAFE
 YES
 ↓
Danger zone?
 YES → DANGER
 NO
 ↓
Warning zone?
 YES → WARNING
 NO → SAFE
```

---

## D — UI 담당

기존 overlay 코드를 찾는다.

```bash
grep -R "bbox" -n .
grep -R "overlay" -n .
grep -R "draw" -n .
```

확인:

```text
bbox를 어디서 그리는가
text drawing 지원 여부
freespace 색상은 어디서 입히는가
lane drawing 함수는 무엇인가
```

Day 1에는 변경하지 않는다.

---

## E — System 담당

전체 실행 흐름을 추적한다.

```text
main
 ↓
camera init
 ↓
V4L2
 ↓
thread create
 ↓
render
 ↓
NPU
 ↓
post-process
 ↓
Wayland
 ↓
cleanup
```

그리고 thread 관련 코드 검색:

```bash
grep -R "pthread_create" -n .
grep -R "pthread_mutex" -n .
grep -R "pthread_cond" -n .
```

---

# 13:00–15:00

## 기존 sample 빌드

전체 SDK를 매번 빌드하지 않는다.

문서에서도 application에는 별도 `build.sh`가 제공되며 빌드 결과가 `output/user_data`로 복사된다고 되어 있다. 

예:

```bash
cd applications
./build.sh
```

빌드 성공하면 결과 확인.

```bash
ls ../output/user_data/applications
```

---

# 15:00–17:00

## 보드에서 baseline 실행

가능하면 NFS를 추천한다.

문서상 NFS 실행 흐름은:

```bash
mount -t nfs <PC_IP>:/home/... /mnt -o nolock
cd /mnt/user_data/applications
./run_app_xxx.sh
```

형태로 제공된다. 

NFS가 이미 세팅되어 있지 않으면 **첫날 NFS 설정 때문에 3시간 쓰지 말고 기존 방식 그대로 사용**해도 된다.

목표는 단 하나.

```text
Camera 화면 보임
Object bbox 보임
Freespace 보임
Lane 보임
```

스크린샷과 동영상 10초 저장.

**이게 Day 1 보험이다.**

---

# 17:00–18:00 — 전체 팀 회의

화이트보드에 실제 코드 기준으로 이것을 완성한다.

```text
              실제 파일 / 함수

Camera      → __________
V4L2        → __________
render      → __________
cnn_task    → __________
NPU         → __________
pp_task     → __________
Detection   → __________
Freespace   → __________
Lane        → __________
Wayland     → __________
```

그리고 Day 2부터 쓸 **공통 데이터 구조를 확정한다.**

---

# Day 1 종료 조건

아래 5개가 다 되어야 한다.

```text
[ ] 기존 Trichimera demo 실행
[ ] 빌드 방법 확정
[ ] 실행 방법 확정
[ ] AI output 데이터 위치 확인
[ ] 수정할 주요 파일/함수 위치 확인
```

하나라도 안 됐으면 밤에 새 기능 개발하면 안 된다.

---

# DAY 2 — AI 결과를 우리 코드로 빼내기

Day 2 목표:

> SDK 내부 데이터 → 우리 `ADASResult`

---

## 09:00–10:00

공통 header 작성.

예:

```cpp
#define MAX_OBJECTS 32

typedef struct {
    int class_id;
    float confidence;

    int x1;
    int y1;
    int x2;
    int y2;
} ObjectInfo;

typedef struct {
    int x;
    int y;
} Point2D;
```

```cpp
typedef struct {
    Point2D points[64];
    int count;
} LaneLine;
```

```cpp
typedef struct {
    LaneLine left;
    LaneLine right;
} LaneInfo;
```

그리고:

```cpp
typedef enum {
    RISK_SAFE,
    RISK_WARNING,
    RISK_DANGER
} RiskLevel;
```

최종:

```cpp
typedef struct {
    uint64_t frame_id;

    ObjectInfo objects[MAX_OBJECTS];
    int object_count;

    LaneInfo lane;

    RiskLevel risk;
} ADASResult;
```

실제 freespace 표현은 **Day 1 확인 결과에 맞게 바꾼다.**

---

# A — AI 담당

post-processing 결과를 기존 overlay로 바로 보내기 전에

```cpp
AIResult ai_result;
```

형태로 복사하는 adapter 작성.

목표:

```cpp
bool getTrichimeraResult(AIResult* out);
```

콘솔 테스트:

```text
frame=121
objects=4
car 0.92 (212,83)-(302,174)
person 0.81 ...
lane_left=...
lane_right=...
```

---

# B — Geometry 담당

AI 없이 가짜 데이터로 함수 구현.

```cpp
Point2D getBottomCenter(const ObjectInfo& obj);
```

```cpp
bool pointInPolygon(
    Point2D p,
    const Point2D* polygon,
    int count);
```

```cpp
bool isInEgoLane(
    Point2D p,
    const LaneInfo& lane);
```

그리고 freespace 표현이 mask라면:

```cpp
bool isOnFreespace(
    Point2D p,
    const uint8_t* mask,
    int width,
    int height);
```

---

# C — Risk 담당

AI와 관계없이 mock input으로 구현.

```cpp
RiskLevel evaluateRisk(
    const ObjectInfo& obj,
    bool on_freespace,
    bool in_ego_lane,
    int image_height);
```

처음에는 아주 단순하게:

```cpp
float y_ratio =
    (float)obj.y2 / image_height;
```

예:

```text
0.00 ~ 0.55 → SAFE
0.55 ~ 0.78 → WARNING
0.78 ~ 1.00 → DANGER
```

숫자는 나중에 실제 영상에 맞춰 tuning.

---

# D — UI 담당

우선 기존 bbox overlay를 건드리지 않고 추가 요소만 준비.

```cpp
drawRiskText();
drawRiskZone();
drawPerformanceText();
```

첫 테스트는:

```text
SAFE
```

문자 하나만 화면에 나와도 성공.

---

# E — System 담당

공유 상태 작성.

```cpp
typedef struct {
    ADASResult result;

    pthread_mutex_t mutex;

    bool running;
} SharedState;
```

getter/setter 형태 추천.

```cpp
void updateADASResult(...);
void copyADASResult(...);
```

다른 사람이 mutex 직접 만지지 않게 하는 게 좋다.

---

# Day 2 종료 조건

```text
[ ] AI 결과를 console로 읽을 수 있음
[ ] geometry 함수 독립 테스트 완료
[ ] risk engine mock test 완료
[ ] 화면에 임의의 SAFE 표시 가능
[ ] shared state 준비 완료
```

---

# DAY 3 — 진짜 위험 판단 연결

오늘부터 프로젝트가 “우리 것”이 된다.

## 오전

A+B 통합:

```text
Detection
   ↓
bottom center
```

화면이나 로그에 기준점 출력.

예:

```text
CAR bbox
    │
    ▼
bottom=(273,281)
```

다음:

```text
bottom center
 ↓
freespace lookup
```

콘솔:

```text
car #0: freespace=1
car #1: freespace=0
```

---

## 오후

lane 연결.

```text
car #0
freespace = YES
ego_lane  = YES

car #1
freespace = YES
ego_lane  = NO
```

그 다음 risk.

```text
car #0 → WARNING
car #1 → SAFE
```

최종적으로 여러 객체 중 가장 높은 위험도를 차량 상태로 사용하면 된다.

```cpp
global_risk =
    max(global_risk, object_risk);
```

위험도 순서:

```text
SAFE < WARNING < DANGER
```

---

# Day 3 테스트 시나리오

반드시 표 만들어서 한다.

| 상황                  | 기대      |
| ------------------- | ------- |
| 도로 밖 차량             | SAFE    |
| 옆 차선 차량             | SAFE    |
| 내 차선 먼 차량           | SAFE    |
| 내 차선 중거리 차량         | WARNING |
| 내 차선 가까운 차량         | DANGER  |
| 가까워도 freespace 밖 객체 | SAFE    |

이걸 발표 자료에도 그대로 쓸 수 있다.

---

# DAY 4 — 시스템 프로그래밍 + UI

여기가 수업 연계 점수를 만드는 날이다.

## Thread

최소 구조:

```text
Main Thread
 │
 ├── Camera / render
 │
 ├── NPU worker
 │
 └── Risk processing
```

기존 SDK가 이미 thread를 사용한다면 억지로 구조를 뜯지 말고, **기존 구조 안에서 별도 risk worker를 둘 가치가 있는지 판단**한다.

중요한 건 “pthread를 썼다”가 아니라 **왜 필요한지 설명할 수 있어야 한다.**

---

## Mutex

공유 결과:

```cpp
pthread_mutex_lock(&state->mutex);

state->result = result;

pthread_mutex_unlock(&state->mutex);
```

UI:

```cpp
pthread_mutex_lock(&state->mutex);

local = state->result;

pthread_mutex_unlock(&state->mutex);
```

UI가 mutex 잡은 채로 그림 그리면 안 된다.

**copy하고 바로 unlock.**

이런 부분이 시스템 프로그래밍에서 꽤 좋은 발표 포인트다.

---

# Signal 처리

E 담당.

```cpp
static volatile sig_atomic_t g_running = 1;

void handle_sigint(int signo)
{
    g_running = 0;
}
```

```cpp
signal(SIGINT, handle_sigint);
```

종료 흐름:

```text
Ctrl+C
 ↓
running = 0
 ↓
worker 종료
 ↓
pthread_join
 ↓
camera/NPU/Wayland cleanup
 ↓
exit
```

---

# 성능 측정

`clock_gettime()` 사용.

```cpp
struct timespec begin, end;

clock_gettime(CLOCK_MONOTONIC, &begin);

/* processing */

clock_gettime(CLOCK_MONOTONIC, &end);
```

측정:

```text
NPU inference
Risk processing
전체 FPS
```

화면 예:

```text
FPS       : 29.8
NPU       : 16.4 ms
RISK      : 0.08 ms
```

---

# Event log

DANGER가 SAFE로 바뀔 때마다 쓰는 게 아니라, **상태 변화 시점만 기록**하는 게 좋다.

```text
SAFE → WARNING
WARNING → DANGER
DANGER → WARNING
```

예:

```text
[15:42:31.128] frame=1832 car confidence=0.91 WARNING
[15:42:32.417] frame=1871 car confidence=0.93 DANGER
```

이것도 시스템 프로그래밍 요소가 된다.

---

# DAY 4 종료 화면

목표는 대략 이렇다.

```text
┌────────────────────────────────────┐
│ FPS 29.8    NPU 16.2ms RISK 0.1ms │
│                                    │
│                 ┌──────┐           │
│                 │ CAR  │           │
│                 └──●───┘           │
│                                    │
│         ╲                  ╱       │
│          ╲     SAFE       ╱        │
│           ╲              ╱         │
│            ╲ WARNING    ╱          │
│             ╲──────────╱           │
│              ╲ DANGER ╱            │
│               ╲______╱             │
│                                    │
│             ⚠ DANGER               │
└────────────────────────────────────┘
```

---

# DAY 5 — 기능 추가 금지

이날은 새로운 기능 넣지 않는다.

오전에는 integration test만 한다.

## 09:00–11:00

테스트 케이스 10회 반복.

```text
부팅
실행
카메라
NPU
risk
Ctrl+C
재실행
```

반드시 **재실행** 테스트까지 한다.

첫 실행은 되는데 resource cleanup이 안 되어 두 번째부터 죽는 경우가 있기 때문이다.

---

# 11:00–12:00

성능 수집.

예:

```text
Average FPS
Average inference latency
Risk engine latency
CPU utilization
```

CPU 사용률까지 하면:

```bash
top
```

정도로 충분하다.

프로파일링까지 할 필요 없다.

---

# 13:00–14:00

데모 영상 촬영.

최소 두 개 저장.

```text
demo_good.mp4
demo_backup.mp4
```

발표 당일 카메라가 안 잡혀도 망하지 않게 한다.

---

# 14:00–16:00 발표 준비

발표는 이 순서가 제일 깔끔하다.

### 1. 문제

```text
Object Detection만으로는
실제 충돌 위험을 판단할 수 없다.
```

### 2. 입력

```text
Detection
Freespace
Lane
```

### 3. Fusion

```text
bottom-center
        ↓
freespace
        ↓
ego lane
        ↓
risk zone
```

### 4. 시스템 구조

```text
V4L2
NPU
pthread
mutex
Wayland
```

### 5. 결과

```text
SAFE
WARNING
DANGER
```

---

# 5명이 발표할 때 역할도 그대로 가져가면 된다

A:

> APACHE6 NPU와 Trichimera integration

B:

> Segmentation 및 lane 결과를 이용한 spatial geometry

C:

> 통합 위험 판단 알고리즘

D:

> Wayland real-time visualization

E:

> Linux thread synchronization, signal handling 및 전체 integration

이러면 “누가 뭘 했는지”도 명확하다.

---

# 최우선 구현 순서

5일 동안은 이 순서를 절대 바꾸지 않는 걸 추천한다.

```text
1. 기존 demo 실행

2. Detection 결과 접근

3. Freespace 결과 접근

4. Lane 결과 접근

5. bottom-center

6. freespace 판정

7. ego-lane 판정

8. SAFE/WARNING/DANGER

9. 화면 표시

10. mutex / signal / timing

11. logging
```

**10번보다 1~9번이 먼저다.**

시스템 프로그래밍 요소를 보여주겠다고 첫날부터 thread 구조를 뜯으면 프로젝트가 흔들릴 가능성이 높다.

---

## 첫날 팀이 실제로 적어둘 체크리스트

이건 그대로 복사해서 쓰면 된다.

```text
[APACHE6 ADAS - DAY 1]

□ 보드 정상 부팅
□ 카메라 정상 연결
□ 기본 Wayland camera sample 실행
□ 기본 Wayland NPU sample 실행
□ Trichimera 모델 위치 확인
□ Trichimera 설정 ini 위치 확인

□ application build 성공
□ board deployment 방법 확인
□ 재빌드 → 실행 반복 가능

□ main() 위치
□ render() 위치
□ cnn_task 위치
□ pp_task 위치
□ V4L2 dequeue 위치
□ Wayland 출력 위치

□ Detection 데이터 구조 확인
□ Freespace 데이터 구조 확인
□ Lane 데이터 구조 확인

□ Git repository 생성
□ main baseline commit
□ 5개 feature branch 생성

□ baseline 실행 영상 저장
```

그리고 **Day 1이 끝났을 때 baseline commit에 태그를 하나 남겨.**

```bash
git tag baseline-working
```

이건 정말 유용하다. 이후 누군가 코드를 망가뜨려도 금요일에 최소한 돌아가는 SDK demo로 즉시 돌아갈 수 있기 때문이다.

마지막으로 APACHE6 쪽에서 이 프로젝트가 잘 맞는 이유도 분명하다. 보드는 카메라 MIPI CSI, NPU, GPU, V4L2, Wayland/DRM 파이프라인뿐 아니라 UART, CAN, I2C, SPI, PWM 등의 peripheral도 제공한다.  다만 **이번 5일 MVP에서는 peripheral까지 욕심내지 말고 Linux application pipeline을 제대로 구현하는 쪽이 훨씬 낫다.** 나중에 시간이 남으면 LED나 CAN을 “확장 가능 기능”으로 발표하면 충분하다.

다음으로는 내가 **실제로 팀이 만들 C/C++ 파일 구조와 `struct`, 함수 prototype을 미리 설계해서, Day 1 끝난 뒤 다섯 명이 그대로 나눠 코딩할 수 있는 수준의 뼈대 코드**까지 만들어줄 수 있어.
