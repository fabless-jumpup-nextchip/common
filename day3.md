
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
