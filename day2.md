
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
