---
title: 스케줄러
date: 2026-02-21 21:59:00 +0900
categories: [멀티스레드, Thread]
tags: [network, thread, scheduler]
description: 스케줄러
---

## OS의 스레드 스케줄링 방식
Windows OS는 아래 방식으로 스레드를 스케줄링 한다.

- 라운드 로빈(round-robin) 방식의 스케줄링
  - (동일한 우선순위의)스레드가 10개가 있다면, 각각의 스레드는 모두 순서대로 1번씩 돌아가면서 실행된다.
- 선점형 우선순위 기반(Preemptive Priority-based)의 스케줄링
  - '선점형' 이라는 것은 OS가 실행중인 스레드를 강제로 정지시킨다는 말이다. OS의 스케줄러가 현재 core에서 다른 스레드를 돌리고싶으면 현재 돌고있는 스레드를 강제로 정지시키고 다른 스레드를 돌린다.
  - '우선순위 기반' 이라는 것은 우선순위가 높은 스레드를 먼저 실행시킨다는 말이다. 예를들면 우선순위가 15인 스레드와 우선순위가 10인 스레드가 있으면 반드시 우선순위가 15인 스레드를 먼저 실행시킨다.

참고: '비선점형' 기반 스케줄링은 현재 core에서 돌고 있는 스레드가 스스로 block 되거나 yield 하지 않으면 다른 스레드를 돌릴 수 없다.

## Quantum
모든 스레드는 스케줄링될 때 할당량(quantum)이라고 불리는 작은 양의 시간을 할당받으며, 이 시간 동안 CPU에서 실행된다.  
스케줄러는 현재 실행중인 모든 스레드가 할당량을 소모했는지 안했는지를 계속해서 확인한다.  
스케줄러는 할당량을 모두 소모한 스레드를 정지시키고 다음 스레드를 돌린다.  

quantum의 양은 정확히 정해져있지 않지만 스레드를 대략 15~20ms 정도 돌릴 수 있는 'CPU 사이클 수'이다.  
할당받은 CPU 사이클 수 만큼 스레드를 돌리면 다음 스레드로 교체된다.  
서버용 OS는 quantum이 더 높은 값으로 설정되어 있는데, 프로세스를 많이 돌리지 않을 것이기 때문이다.  

스레드 실행 도중에 해당 core에서 인터럽트를 처리해야 한다면?  
인터럽트는 수시로 발생하고 즉시 처리되어야 한다.  
인터럽트에는 타이머 인터럽트(Timer Interrupt), 디스크 인터럽트, 네트워크 인터럽트 등이 있다.  
그런데 인터럽트는 현재 스레드의 코드가 아니기 때문에 인터럽트를 처리하는 시간은 할당량에서 차감하지 않는다.  

## 스레드 우선순위
Windows 스레드 우선순위는 프로세스 우선순위 클래스(Process Priority Class)와 상대 스레드 우선순위(Thread Relative Priority)의 조합으로 결정된다.  
스레드의 최종 우선순위 값은 [프로세스 우선순위 클래스 값 + 상대 스레드 우선순위 값] 이다.  
최종적인 우선순위의 값은 0 ~ 31 사이이며, 값이 높을수록 우선순위가 높다.  
우선순위가 높은 스레드는 우선순위가 낮은 스레드보다 (거의 항상)먼저 실행된다.  

최종적인 우선순위 값에 따른 구분:
1. 실시간(Real-time) 범위 : 16~31
2. 동적(Dynamic) 범위 : 1~15

#### 프로세스 우선순위 클래스
(정확한 값은 windows 버전에 따라 다를 수 있음)

1. 실시간(Realtime) : 24
  - 중요한 커널 작업용
2. 높음(High) : 13
3. 보통 이상(Above Normal) : 10
4. 보통 (Normal) : 8
  - 일반적인 프로그램의 우선순위
5. 보통 이하(Below Normal) : 6
6. 유휴 상태(Idle) : 4

#### 상대 스레드 우선순위
하나의 프로세스 내에 여러 스레드가 있을 때 스레드들 간의 우선순위를 나타낸다.  

(정확한 값은 windows 버전에 따라 다를 수 있음)
1. Time critical : 프로세스 우선순위 클래스와 상관없이 스레드의 우선순위가 15 또는 31로 설정된다.
2. Highest : +2
3. Avove normal : +1
4. Normal : +0
5. Below normal : -1 
6. Lowest : -2
7. Idle : 프로세스 우선순위 클래스와 상관없이 스레드의 우선순위가 1로 설정된다.

#### 스레드 우선순위 부스팅(thread priority level boosting)
OS는 I/O 이벤트에 응답하거나 윈도우 메시지나 디스크를 읽기 위해 스레드의 우선순위 레벨을 상승시키기도(thread priority level boosting) 한다.  
예를들면 사용자가 키보드를 누르면 시스템은 WM_KEYDOWN 메시지를 스레드의 메시지 큐에 삽입한다.  
이 경우 스레드는 처리해야 할 작업이 생겼으므로 스케줄 가능 상태가 된다.  
그리고 키보드 드라이버는 OS에게 스레드의 우선순위 레벨을 임시적으로 상승시켜 줄 것을 요청한다.  
그러면 OS가 스레드의 우선순위 레벨을 2만큼 상승시킨다.  

스레드는 1 quantum 시간 동안만 상승된 우선순위로 실행된다.  
1 quantum 시간이 지나면 OS는 스레드의 우선순위 레벨을 1 감소시킨다.  
또한번의 1 quantum 시간이 지나면 OS는 스레드의 우선순위 레벨을 1 감소시킨다.  
우선순위를 얼마나 상승시킬지는 디바이스 드라이버가 결정한다.  
그런데 우선순위 부스팅은 우선순위 레벨이 1~15 사이인 스레드에 대해서만 적용된다.  
그래서 이 범위의 우선순위 레벨을 dynamic priority range 라고 한다.  
또한 OS는 실시간 우선순위 범위(15를 초과하는)로는 우선순위를 상승시키지 않는다.  

## ready queue
스케줄러는 스레드 스케줄링을 위해 ready queue를 가진다.  
ready queue는 CPU 논리 core 1개당 우선순위 개수만큼 존재한다.  
ready queue에는 실행되기를 기다리는 스레드의 TCB(Thread Control Block. KTHREAD block 등의 스레드 컨텍스트를 말함)가 저장되어 있다.  
스케줄러는 ready queue에서 우선순위가 가장 높은 TCB 1개를 꺼내서 core에 입력하여 스레드를 실행시킨다.  
스레드가 실행되다가 자신의 quantum을 다 쓰면 스케줄러가 해당 스레드를 정지시키고 ready queue의 가장 뒤에 삽입한다.  

![ready queue](/assets/img/posts/2026-02-21-Network-Mutlthread-scheduler-img.png)

스케줄러는 TCB를 다른 core의 ready queue에 넣을수도 있다.  
예를들면 core1의 ready queue에는 TCB가 많이 있는데, core2의 ready queue에 TCB가 없다면 TCB중 일부를 core2의 ready queue에 넣어줄 수도 있다.  

## timer interrupt
timer interrupt는 메인보드가 주기적으로 발생시키며, timer interrupt가 발생하면 OS는 시스템 시간을 갱신하고, 스케줄러를 실행한다.  

## 스케줄러와 dispatcher
스케줄러는 현재 실행중인 스레드의 quantum 할당량 등을 확인하여 현재 스레드가 다른 스레드로 교체되어야 하는지를 확인한다.  
만약 교체되어야 하면 dispatcher 인터럽트를 발생시켜서 dispatcher를 실행시킨다.  
dispatcher는 실제로 스레드를 교체하는(context switching 등) 작업을 한다.  

스케줄러가 실행되는 시점은 다음과 같다:
1. 스레드가 생성되거나 종료될 때
2. 실행 중이던 스레드가 block 될때
3. timer interrupt가 발생했을 때

dispatcher가 실행되는 시점은 다음과 같다:
1. (스케줄러 등에 의해)dispatcher interrupt가 발생했을 때

스케줄러는 다음 일들을 한다:
1. 실행중인 스레드가 quantum을 다 썼는지 확인한다.
2. quantum을 다 사용한 스레드가 있거나, 깨어나야 하는 스레드 등이 있으면 dispatcher interrupt를 발생시킨다.

dispatcher는 다음 일들을 한다:
1. 현재 실행중인 스레드의 컨텍스트를 TCB(Thread Control Block)에 저장한다.
2. TCB를 스레드 우선순위에 해당하는 ready queue의 맨 뒤에 삽입한다(이 때 다른 core에 여유가 있다면 다른 core의 ready queue에 삽입할 수도 있다).
3. 가장 높은 우선순위의 ready queue에서 TCB를 하나 꺼내서 core에 입력하여 스레드가 실행되도록 한다.
