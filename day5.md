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
