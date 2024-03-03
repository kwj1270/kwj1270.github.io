---
title: ZGC
author: woozi
date: 2024-03-03 12:00:00 +0800
categories: [Blogging, Tech]
tags: [JVM]
---

# 개요

- Low Latency: GC 일시 정지 시간이 10ms 미만이다.
- Scalable: heap 사이즈나 라이브셋의 사이즈가 커져도 일시 정지 시간이 늘어나지 않는다.

# 목표

- GC 일시 정지 시간이 10ms를 초과하지 않아야 한다.
- 비교적 작은(수백 MB) 크기에서 매우 큰(수 TB) 사이즈의 heap을 다룰 수 있어야 한다.
- G1GC 보다 애플리케이션 처리율이 15% 이상 떨어지지 않을 것.
- colored pointers, load barriers를 사용하여 미래의 GC를 위한 기능/최적화 기반을 마련한다.
- 최초 지원 플랫폼은 Linux/x64.

# Region

![ZGC Region](/assets/img/blog/zgc.png)

- ZGC 는 ZPage 라고 하는 Region 을 나누어 Heap 메모리를 관리한다.(G1GC와 유사)
- ZPage 는 동적으로 크기가 지정이 되며 2MB의 배수가 동적으로 생성 및 소멸될 수 있다.
- ZPage 는 G1GC 와 다르게, 메모리 크기 3단계로 구분되어 사용된다.
  - Small : 2MB
  - Medium : 32MB
  - Large : (N X 2 MB)

# Phases of Z Garbage Collection

![ZGC Phase](/assets/img/blog/phase.jpeg)

- Pause Mark Start
  - Mark Start STW
  - Concurrent Mark/Remap
- Pause Mark End
  - Mark End STW
  - Concurrent Pereare for Relocate
- Pause Relocate Start
  - Relocate Start STW
  - Concurrent Pereare for Relocate

## Pause Mark Start

1. Mark Start STW : ZGC의 Root에서 가리키는 객체에 Mark 표시한다.
2. Concurrent Mark/Remap : 객체의 참조를 탐색하면서 거쳐가는 객체에 Mark 표시한다.

`Mark Start STW` 단계는
객체를 참조하는 Root Set 을 찾은 다음 Marking 하는 작업이다.
Root Set 은 상대적으로 적기 때문에 매우 짧은 STW 를 가진다는 특징을 가지고 있다.

`Concurrent Mark/Remap` 단계는 Application 과 동시에 수행되며
Marking 된 Root set 으로부터 객체 간의 참조 관계(그래프)를 추적하여
접근한 모든 객체를 Marking 하는 작업이다.(Marked bit check)

## Pause Mark End

1. Mark End STW : 새롭게 들어온 객체들에 대해 Mark 표시
2. Concurrent Pereare for Relocate : 재배치하려는 영역을 찾아 Relocation Set에 배치

`Mark End STW` 는 point 들을 동기화하며
`Concurrent Mark/Remap` 단계 동안 새로 들어온 객체들에 대해서 Marking 하는 작업이다.

`Concurrent Pereare for Relocate` 는
재 배치하려는 영역을 찾아 Relocation Set에 배치하며
Week, Phantom Reference와 같은 일부 edge case를 확인하고 정리한다.

## Pause Relocate Start

1. Relocate Start STW : 모든 Root 참조의 재배치를 진행하고 업데이트
2. Concurrent Relocate: **relocation set** 과 **Load Barriers** 를 사용하여 모든 객체를 재배치 및 참조 수정

**Load Barriers 를 활용한 재배치 및 참조 수정 과정**

1. mark pointer의 색이 나쁜 경우
   mark, relocate, re-mapping을 진행하여 좋은 상태 (색상)로 변경하는 작업을 진행한다.
2. mark pointer 의 색이 좋은 경우 그대로 작업을 진행한다.
3. Remap bit가 1인 경우 바로 참조 값을 반환하며
   그렇지 않은 경우에는 참조된 개체가 Relocation Set에 있는지 확인한다.
4. Set에 없는 경우 Remap bit를 1로 설정한다. (재 배치 되었음을 의미하기에)
5. Set에 있는 경우에는 Relocation 하고
   forwarding table에 해당 정보를 기록한 뒤 Remap bit를 1로 설정한다.
6. 참조 값을 반환한다.

## **colored pointers**

![Colored Pointers](/assets/img/blog/colored-pointers.png)

- ZGC가 객체를 찾아낸 뒤, 마킹하고, 재 배치하는 등의 작업을 지원한다.
- 객체 포인터의 메모리 공간을 활용하여 **객체의 상태 값을 저장하고 사용한다.**
- 해당 알고리즘 방식이 64bit 메모리 공간을 필요로 하기 때문에
  32 bit 기반의 플랫폼에서는 사용이 불가능하다.

**bits 구분**

- 18비트: **Unusedbits**
- 1비트: **Finalizable**
  - Finalizer(Finalize queue)을 통해서 참조되는 객체로
    해당 pointer가 Mark 되어 있다면 non-live Object이다.
- 1비트: **Remapped**
  - 해당 객체의 재배치 여부를 판단하는 pointer이며,
    해당 Bit의 값이 1이라면 최신 참조 상태임을 의미한다.
- 1비트 & 1비트 : **Marked1 & Marked0**
  - 해당 객체가 Live된 상태 인지 확인하는 여부이다.
  - Load Barrier에 의해서도 사용되기도 한다.
- 42비트: **Object Address**

## **load barriers**

![load-barreiers1](/assets/img/blog/load-barriers1.png)

- JIT Compiler 에 의해 주입되는 코드의 일종이다.
- colored pointers 를 기준으로 잘못되거나 잘못된 색상으로 검출될 경우 복구 조치를 취한다.

![load-barreiers2](/assets/img/blog/load-barriers2.png)

- load barrier 는 Thread가 Stack으로 Heap Object 참조 값을 불러올 때 실행된다.(주로 메서드 내)

![load-barreiers3](/assets/img/blog/load-barriers3.png)

- load barrier 를 통해 참조가 연결되는 객체의 Meta bits 를 확인하는 작업을 진행한다.
- Meta bits 를 확인할 때, 잘못된 색상으로 검출될 경우 복구 조치를 취한다.

![load-barreiers4](/assets/img/blog/load-barriers4.png)

`ZGC` 는 메모리를 재배치하는 과정에서 **bit 를 사용하여 STW 없이 재배치를 한다.**
이때 RemapMark와 RelocationSet을 확인하면서 참조와 Mark를 업데이트한다.

# **Forwarding table**

![fowrading-table](/assets/img/blog/fowarding-table.png)

Relocation 대상인 객체의 현재 참조 값과 변경 후 참조 값을 기록하는 일종의 Mapping Table 이다.
이를 이용하여 현재 Relocation 된 객체를 바로 접근하고 참조할 수 있다.

# 그림으로 살펴보자

## 기본 예시 환경

![zgc-phase1](/assets/img/blog/zgc-phase1.png)

## Pause Mark Start

![zgc-phase2](/assets/img/blog/zgc-phase2.png)

![zgc-phase3](/assets/img/blog/zgc-phase3.png)

- Root 로 부터 참조되고 있는 객체들에 한하여 Marking 을 진행한다.

## Conccurrent Mark

![zgc-phase4](/assets/img/blog/zgc-phase4.png)

![zgc-phase5](/assets/img/blog/zgc-phase5.png)

## Pause Mark End

![zgc-phase6](/assets/img/blog/zgc-phase6.png)

## Concurrent Prepare for Relocate

![zgc-phase7](/assets/img/blog/zgc-phase7.png)

![zgc-phase8](/assets/img/blog/zgc-phase8.png)

## Pause Relocate Start

![zgc-phase9](/assets/img/blog/zgc-phase9.png)

## Concurrent Relocate

![zgc-phase10](/assets/img/blog/zgc-phase10.png)

![zgc-phase11](/assets/img/blog/zgc-phase11.png)

![zgc-phase12](/assets/img/blog/zgc-phase12.png)

![zgc-phase13](/assets/img/blog/zgc-phase13.png)

## GC Cycle Completed

![zgc-phase14](/assets/img/blog/zgc-phase14.png)

## Pause Mark Start (Second Cycle)

![zgc-phase15](/assets/img/blog/zgc-phase15.png)

## Concurrent Prepare for Relocate (Second Cycle)

![zgc-phase16](/assets/img/blog/zgc-phase16.png)

# 아직 다루지 못한 내용(추후 추가 예정)

1. Striped Mark
2. Heap Address Space
