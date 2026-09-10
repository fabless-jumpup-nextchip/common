5일 일정은 **이미 있는 Trichimera 데모 위에 risk 모듈만 얹는 것**을 기준으로 다시 짰습니다. Camera / NPU / Overlay는 만들지 않습니다.

---

# 목표

금요일에 보드에서 이 프로그램 하나가 돌아가면 끝입니다.

```text
Camera → V4L2 → Trichimera @ NPU
                ↓
     Detection + Freespace + Lane   ← 이미 있음
                ↓
           Risk Engine              ← 이번 주 본체
                ↓
      SAFE / WARNING / DANGER
                ↓
         Wayland Overlay            ← 문구/존만 추가
```

만들지 않는 것:

- NPU init / `cnn_task` / Trichimera 후처리
- bbox, freespace mask, lane polyline 그리기
- 새 NPU worker, risk worker 스레드
- `USE_ADAS_LD` (차선이탈, 다른 주제)
- CAN / LED / peripheral

---

# 역할과 파일 소유권

| 팀원 | 담당 | 소유 파일 | 만지지 말 것 |
|---|---|---|---|
| A | AI 추출 | `nc_adas_extract.c/.h` | 후처리, NPU, `wayland_npu_app.c` |
| B | Geometry | `nc_adas_geometry.c/.h` | OpenGL, V4L2 |
| C | Risk | `nc_adas_risk.c/.h` | OpenGL, NPU |
| D | UI | `nc_adas_ui.c/.h` | NPU, 후처리 |
| E | 통합 | `wayland_npu_app.c`, `Makefile` | 판단 수식 |

공통 헤더 `nc_adas_types.h`는 Day 1 저녁에 `main`에 넣고, 이후에는 전원 합의 없이 바꾸지 않습니다.

위치:

```text
applications/common/nc_app_modules/adas/risk_judge/
  nc_adas_types.h
  nc_adas_extract.c / .h
  nc_adas_geometry.c / .h
  nc_adas_risk.c / .h
  nc_adas_ui.c / .h
```

언어는 **C**입니다.

---

# Git

```text
main          ← 항상 보드에서 실행 가능
짧은 개인 브랜치  ← 하루 단위로 merge
```

1. `main`은 매일 저녁 실행 가능한 상태
2. 기능 하나 끝날 때마다 merge (브랜치를 금요일까지 유지하지 않음)
3. `wayland_npu_app.c`는 E만 수정

Day 1 종료 시:

```bash
git tag baseline-working
```

---

# DAY 1 — baseline 확보 + 인터페이스 고정

개발하는 날이 아닙니다. **데모가 살아 있고, 타입 헤더가 `main`에 들어가 있어야** Day 2가 성립합니다.

## 09:00–10:00 전원

검색은 `applications/`만 합니다.

확인만 하고 넘어갑니다.

```text
wayland_npu_app.c:663   render()
wayland_npu_app.c:980   cnn_task()
wayland_npu_app.c:1102  nc_cnn_postprocess_task
Makefile                CNN_NETWORK_SETTING = TRICHIMERA
DETECT_NETWORK=0
SEGMENT_NETWORK=4
LANE_NETWORK=8
NPU 입력 640x384
화면 1920x1080
```

`USE_ADAS_LD`는 주석 그대로 둡니다.

## 10:00–13:00

**A**  
후처리 결과를 읽기만 합니다. 수정하지 않습니다.

```text
det  → pp_result_buf.cnn_result.class_objs[]
seg  → pp_result_buf.cnn_result.seg  (픽셀 마스크)
lane → pp_result_buf.cnn_result.lane_det[]
bbox → stBBox {x, y, w, h}   (x1y1x2y2 아님)
class → per, car, bus, tru, cycle, mot
```

**B**  
mask가 polygon이 아님을 확인하고, 좌표 세 장을 종이에 그립니다.

```text
NPU 640x384
mask (후처리 해상도)
overlay 1920x1080
```

**C**  
PC에서 risk 흐름만 확정. SDK 안 건드림.

```text
객체 bottom-center
 → freespace 아니면 SAFE
 → ego lane 아니면 SAFE
 → 가까우면 DANGER / 중거리 WARNING / 아니면 SAFE
```

**D**  
기존 그리기 위치를 찾기만 합니다. `nc_draw_gl_npu`, `nc_opengl_draw_text`. 오늘은 문구를 넣지 않습니다.

**E**  
빌드·실행 경로를 팀에 공지합니다.

```bash
cd applications
./build.sh          # 또는 cd wayland_npu_app && make
# 결과: output/user_data/applications/app_wayland_npu

# 보드
./run_app_wayland_npu.sh
```

NFS가 안 되어 있으면 첫날 NFS를 새로 깔지 말고 기존 배포 방식을 씁니다.

## 13:00–17:00 전원 — 보드 실행

성공 기준은 화면입니다.

```text
카메라 보임
객체 bbox 보임
freespace 보임
lane 보임
```

추가로 **A+B가 같이** 마스크 픽셀 값 하나를 로그로 확인합니다. INI는 `background=0, freespace=1`인데, 그리기 코드와 값이 어긋날 수 있습니다. 이 숫자를 저녁에 헤더 주석으로 고정합니다.

10초 영상 저장.

## 17:00–18:00 전원 회의

`nc_adas_types.h`를 확정하고 E가 `main`에 커밋합니다.

```c
#define ADAS_MAX_OBJECTS 32
#define ADAS_MAX_LANE_POINTS 50
#define ADAS_MAX_LANES 10

typedef enum {
    RISK_SAFE = 0,
    RISK_WARNING,
    RISK_DANGER
} AdasRiskLevel;

typedef struct {
    int class_id;
    float confidence;
    float x, y, w, h;   /* SDK stBBox와 동일 */
} AdasObject;

typedef struct {
    int x, y;
} AdasPoint;

typedef struct {
    AdasPoint points[ADAS_MAX_LANE_POINTS];
    int point_cnt;
    int lane_class;
} AdasLane;

typedef struct {
    uint64_t frame_id;
    int width;          /* overlay width  */
    int height;         /* overlay height */

    AdasObject objects[ADAS_MAX_OBJECTS];
    int object_count;

    const uint8_t *freespace_mask;
    int mask_width;
    int mask_height;
    uint8_t freespace_value;   /* Day 1에 확인한 값 */

    AdasLane lanes[ADAS_MAX_LANES];
    int lane_count;

    AdasRiskLevel global_risk;
} AdasResult;
```

함수 계약도 같이 잠급니다.

```c
/* A */ int  adas_extract(const pp_result_buf *det,
                         const pp_result_buf *seg,
                         const pp_result_buf *lane,
                         AdasResult *out);

/* B */ AdasPoint adas_bottom_center(const AdasObject *obj);
        int  adas_on_freespace(AdasPoint p, const AdasResult *r);
        int  adas_in_ego_lane(AdasPoint p, const AdasResult *r);

/* C */ AdasRiskLevel adas_evaluate_object(...);
        AdasRiskLevel adas_evaluate_frame(AdasResult *r);

/* D */ void adas_draw_overlay(const AdasResult *r, ...);
```

E는 Makefile에 `risk_judge` 소스를 넣는 자리만 만들어 두고, `render()` 연결은 Day 2에 합니다.

### Day 1 종료 조건

```text
[ ] app_wayland_npu 실행, bbox/seg/lane 확인
[ ] 빌드/배포 방법 확정
[ ] freespace 픽셀 값 확정
[ ] nc_adas_types.h main 커밋
[ ] git tag baseline-working
[ ] baseline 영상 저장
```

하나라도 실패하면 밤에 새 기능 넣지 않습니다.

---

# DAY 2 — 모듈을 따로 만들고, 화면에는 SAFE만

목표: 다섯 모듈이 **서로 기다리지 않고** 컴파일되게 만드는 날입니다. 진짜 fusion은 Day 3입니다.

## A

NPU를 부르지 않습니다. `render()`가 이미 읽는 버퍼 3개를 flatten해서 `AdasResult`에 넣습니다.

콘솔:

```text
frame=121 objects=4
car 0.92 xywh=(212,83,90,91)
mask=640x384 value_at_center=1
lanes=2
```

bbox 좌표와 mask 샘플을 **한 줄에** 찍습니다. 좌표가 안 맞으면 여기서 고칩니다.

끝나면 B 좌표 작업을 돕습니다.

## B (AI 없이 mock)

```c
adas_bottom_center()     /* (x + w/2, y + h) */
adas_on_freespace()      /* mask 샘플. polygon 아님 */
adas_in_ego_lane()       /* 일단 화면 하단 중앙 사다리꼴 stub */
```

PC 단위 테스트 가능하면 그걸로 먼저 검증합니다. Ego-lane 본구현은 Day 3 오전입니다.

## C (mock)

```c
/* y_ratio = obj.y + obj.h  / image_height */
0.00–0.55 SAFE
0.55–0.78 WARNING
0.78–1.00 DANGER
```

`on_freespace==0` 또는 `in_ego_lane==0`이면 무조건 SAFE.

## D

기존 bbox/mask/lane은 그대로 둡니다. 기존 FPS 텍스트 옆에 `SAFE` 한 줄만 그립니다. zone 사다리는 Day 4입니다.

## E

`render()`에서 버퍼 3개를 읽은 뒤, 아직 risk 없이:

1. `adas_extract(...)`
2. stub `AdasResult.global_risk = RISK_SAFE`
3. `adas_draw_overlay(...)`

연결은 E만 합니다. 다른 사람은 `wayland_npu_app.c`에 PR하지 않습니다.

저녁에 `main` merge. 보드는 기존처럼 bbox가 보이고, 좌상단에 `SAFE`가 뜨면 성공입니다.

### Day 2 종료 조건

```text
[ ] extract 결과가 콘솔에 나옴
[ ] bbox 좌표와 mask 샘플이 같은 로그에 있음
[ ] geometry / risk가 mock으로 컴파일됨
[ ] 화면에 SAFE 표시
[ ] main merge, 보드에서 재실행
```

---

# DAY 3 — 진짜 판단 연결

오늘부터 “우리 프로그램”이 됩니다. 기능 추가보다 **표의 5개 상황이 맞는가**가 목표입니다.

## 오전 A+B

1. 각 객체 `bottom-center`를 로그/점으로 표시  
   `car #0 bottom=(273,281)`
2. 그 점이 freespace인지  
   `car #0 freespace=1`
3. 좌표가 틀리면 여기서 하루를 씁니다. risk를 먼저 튜닝하지 않습니다.

이어서 B가 ego-lane 본구현:

- UFLD `lane_det[]`에서 하단 기준 왼쪽/오른쪽 차선 선택
- 두 선 사이를 ego lane으로 봄
- 차선이 1개면 “판단 불가 → SAFE”로 빠져도 됩니다 (과하지 말 것)

## 오후 B+C+E

```text
car #0 freespace=YES ego_lane=YES → WARNING/DANGER
car #1 freespace=YES ego_lane=NO  → SAFE
```

전역 상태는 객체 중 max:

```text
SAFE < WARNING < DANGER
```

D는 글자만 `SAFE` / `WARNING` / `DANGER`로 바꿉니다. 색만 바꿔도 됩니다.

### 반드시 이 표로 찍습니다

| 상황 | 기대 |
|---|---|
| 도로 밖 차량 | SAFE |
| 옆 차선 차량 | SAFE |
| 내 차선 먼 차량 | SAFE |
| 내 차선 중거리 | WARNING |
| 내 차선 가까운 차량 | DANGER |
| 가까워도 freespace 밖 | SAFE |

표가 안 맞으면 threshold를 만지지 말고 **좌표/마스크/차선**을 먼저 의심합니다.

### Day 3 종료 조건

```text
[ ] 표 6행 중 4행 이상 실제 영상에서 재현
[ ] 화면에 현재 risk 문구
[ ] main merge
[ ] 데모 후보 영상 1개
```

4행이 안 되면 Day 4 오전에 이어서 하고, zone 그리기·로그는 줄입니다.

---

# DAY 4 — 수업 점수용 시스템 요소 + UI 보강

**스레드를 추가하지 않습니다.** 이미 `render` / `cnn_task` / `nc_cnn_postprocess_task`가 있습니다. SIGINT, `running`, `pthread_join`, `clock_gettime`도 있습니다.

오늘 할 일:

## E — 설명 가능한 동기화

새 worker 대신 기존 구조를 발표 포인트로 씁니다.

- 후처리 결과는 이미 flip-flop 버퍼로 동기화됨
- UI는 `AdasResult`를 **복사한 뒤** 그린다 (락을 잡고 그리지 않음)
- mutex를 쓰더라도 extract 직후 로컬 복사 한 번이면 충분

Ctrl+C 재실행이 안 되면 cleanup만 보강합니다. 파이프라인은 뜯지 않습니다.

## C+E — 이벤트 로그

매 프레임이 아니라 **상태 변화만**.

```text
[15:42:31.128] frame=1832 car conf=0.91 SAFE → WARNING
[15:42:32.417] frame=1871 car conf=0.93 WARNING → DANGER
```

깜빡이면 같은 상태가 N프레임 유지될 때만 바꾸면 됩니다 (hysteresis).

## A+E — 성능 숫자

기존 FPS 줄에 붙입니다. 새 프로파일러는 없습니다.

```text
FPS 29.8   NPU 16.2ms   RISK 0.1ms
```

`top`으로 CPU 한 줄만 기록해 두면 Day 5 발표용입니다.

## D — zone overlay

시간 남을 때만:

```text
상단 SAFE / 중단 WARNING / 하단 DANGER 사다리꼴
현재 상태 큰 글자
```

기존 bbox/seg/lane보다 위에 얹지 말고, 안 보이면 반투명만 유지합니다.

### Day 4 종료 조건

```text
[ ] Day 3 표가 여전히 맞음
[ ] 상태 변화 로그
[ ] FPS + RISK ms 화면 표시
[ ] Ctrl+C → 재실행 1회 성공
[ ] zone은 있으면 좋고, 없어도 Day 5 진행
```

---

# DAY 5 — 기능 추가 금지

## 09:00–11:00 통합 테스트 10회

```text
부팅 → 실행 → 카메라 → NPU → risk → Ctrl+C → 재실행
```

재실행까지가 필수입니다.

## 11:00–12:00 숫자 수집

```text
Average FPS
NPU latency
Risk latency
CPU (top)
```

## 13:00–14:00 영상

```text
demo_good.mp4
demo_backup.mp4
```

발표 당일 카메라가 없어도 돌아가게 합니다.

## 14:00–16:00 발표

1. Detection만으로는 충돌 위험을 못 본다  
2. 입력: Detection + Freespace + Lane  
3. Fusion: bottom-center → freespace → ego lane → zone  
4. 시스템: V4L2, NPU, 기존 pthread, flip-flop, Wayland  
5. 결과: SAFE / WARNING / DANGER + 표 6행

역할 그대로:

- A: Trichimera 결과 추출  
- B: mask / lane geometry  
- C: risk  
- D: overlay  
- E: 통합, 동기화, signal, 성능

---

# 구현 순서 (이 순서는 유지)

```text
1. 기존 demo 실행
2. types.h 고정
3. Detection 복사
4. Freespace 마스크 복사 + 픽셀 값 확인
5. Lane 복사
6. 좌표 검증 (bbox vs mask)
7. bottom-center
8. freespace 판정
9. ego-lane 판정
10. SAFE/WARNING/DANGER
11. 화면 문구
12. 로그 / hysteresis / timing
```

12보다 6이 먼저입니다. 좌표가 틀린 채로 threshold를 만지면 목요일이 날아갑니다.

---

# 매일 저녁 18:00

```text
[ ] main merge
[ ] 보드에서 한 번 실행
[ ] 오늘 영상 또는 스크린샷
[ ] 내일 막히는 사람 한 명 지정 (보통 B)
```

A는 extract가 끝나는 즉시 B로 붙습니다. D는 Day 2 이후 할 일이 적으니 Day 3 표 촬영, Day 4 존, Day 5 발표 자료를 맡는 편이 맞습니다.
