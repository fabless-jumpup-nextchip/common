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

