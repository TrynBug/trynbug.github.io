---
title: 피터슨 알고리즘의 문제점
date: 2026-02-05 00:18:40 +0900
categories: [CPU, Out-of-Order-Execution]
tags: [cpu, outofordering]
description: 피터슨 알고리즘의 문제점
---

## 1. 피터슨 알고리즘(Peterson's Algorithm)
피터슨 알고리즘은 Lock 없이 2개 스레드를 동기화하는 알고리즘이다.  
스레드 2개에서만 작동하도록 설계되었다.

Peterson's Algorithm
```cpp
// 동기화 플래그 배열
bool flag[2] =
{
    false, // 0번 스레드 진입여부
    false  // 1번 스레드 진입여부
};
int turn = 0;

void Thread0() 
{
    flag[0] = true;  // 0번 스레드가 진입했음을 표시
    turn = 0;
    while (true) 
    {
        if (flag[1] == false)  // 1번 스레드가 사용중이 아니라면 while문을 나가서 임계영역에 진입한다.
            break;
        if (turn != 0)   // flag[1] == true 이어도, turn이 0이 아니라면 1번 스레드가 turn을 1로 바꾼 것이다.
                         // 그렇다면 1번 스레드가 작업을 끝내고 turn=1; 까지 수행했다는 것인데, 나보다는 늦게 while 문에 진입했다는 것이다. 그래서 내가 임계영역에 진입한다.
            break;
    }

    // 임계영역 시작
    // do something..
    // 임계영역 종료
    flag[0] = false;
}

void Thread1() 
{
    flag[1] = true;
    turn = 1;
    while (true) 
    {
        if (flag[0] == false)
            break;
        if (turn != 1)
            break;
    }

    // 임계영역 시작
    // do something..
    // 임계영역 종료
    flag[1] = false;
}
```

피터슨 알고리즘은 이론적으로는 문제가 없지만, 현시점의 CPU에서는 문제가 발생한다.  
현시점의 CPU는 최적화를 위해 명령어들의 순서를 바꾸어 실행(Out-of-Order Execution)할 수 있기 때문이다.  

## 2. 피터슨 알고리즘에 문제가 발생하는 이유

문제 발생 순서:
1. 0번 스레드가 lock을 획득함
2. lock을 해제하기 위해 `flag[0] = false;`로 바꿈

3. 1번 스레드가 lock을 획득함
4. lock을 해제하기 위해 `flag[1] = false;`로 바꿈

5. 0번 스레드와 1번 스레드가 동시에 작동함

6. 코드 상으로는 0번 스레드가 `flag[0] = true;` 을 실행한 다음에 `flag[1]`을 읽어야 하지만,  
Out-of-Order Execution에 의해 `flag[1]`을 읽는 명령어가 먼저 수행되었다.  
0번 스레드는 `flag[1]`(값은 false)을 읽어서 `eax` 레지스터에 저장해두었다.
7. 코드 상으로는 1번 스레드가 `flag[1] = true;` 을 실행한 다음에 `flag[0]`를 읽어야 하지만,  
Out-of-Order Execution에 의해 `flag[0]`를 읽는 명령어가 먼저 수행되었다.  
1번 스레드는 `flag[0]`(값은 false)을 읽어서 `eax` 레지스터에 저장해두었다.

8. 0번 스레드가 `flag[0] = true; g_turn = 0;` 실행
9. 1번 스레드가 `flag[1] = true; g_turn = 1;` 실행

10. 0번 스레드가 `if (flag[1] == false)`를 검사하는데, 이전에 `eax` 레지스터에 넣어둔 값(false)을 사용하여 검사하였다. 
조건이 참이므로 lock 획득
11. 1번 스레드가 `if (flag[0] == false)`를 검사하는데, 이전에 `eax` 레지스터에 넣어둔 값(false)을 사용하여 검사하였다. 
조건이 참이므로 lock 획득

12. 0번 스레드와 1번 스레드가 임계영역에 동시에 진입한다.


## 3. Memory Barrier

Memory Barrier는 메모리 배리어 위쪽 코드와 아래쪽 코드 사이에 Out-of-Order Execution이 발생하지 않도록 하는 기능이다.

```cpp
    MemoryBarrier();
00007FF7B610108D  lock or     dword ptr [rsp],edx  
```

MemoryBarrier는 lock 어셈블리 코드로 변환된다.  
CPU가 lock 명령어를 수행하려면 이전의 모든 명령어가 끝나기를 기다려야 하기 때문에 lock 아래의 명령어가 lock 위의 명령어보다 먼저 실행되지 못하게 하는 효과가 있다.

## 4. Memory Barrier를 사용하여 피터슨 알고리즘의 문제 없애기

Memory Barrier를 사용하여 문제가 없어진 피터슨 알고리즘
```cpp
// 동기화 플래그 배열
bool flag[2] =
{
    false, // 0번 스레드 진입여부
    false  // 1번 스레드 진입여부
};
int turn = 0;

void Thread0() 
{
    flag[0] = true;  // 0번 스레드가 진입했음을 표시
    turn = 0;

    MemoryBarrier(); // 메모리 배리어로 인해 아래의 flag[1] 읽기 작업이 위의 작업보다 먼저 수행되지 못한다.
    
    while (true) 
    {
        if (flag[1] == false)  // 1번 스레드가 사용중이 아니라면 while문을 나가서 임계영역에 진입한다.
            break;
        if (turn != 0)   // flag[1] == true 이어도, turn이 0이 아니라면 1번 스레드가 turn을 1로 바꾼 것이다.
                         // 그렇다면 1번 스레드가 작업을 끝내고 turn=1; 까지 수행했다는 것인데, 나보다는 늦게 while 문에 진입했다는 것이다. 그래서 내가 임계영역에 진입한다.
            break;
    }

    // 임계영역 시작
    // do something..
    // 임계영역 종료
    flag[0] = false;
}

void Thread1() 
{
    flag[1] = true;
    turn = 1;

    MemoryBarrier();  // 메모리 배리어로 인해 아래의 flag[0] 읽기 작업이 위의 작업보다 먼저 수행되지 못한다.

    while (true) 
    {
        if (flag[0] == false)
            break;
        if (turn != 1)
            break;
    }

    // 임계영역 시작
    // do something..
    // 임계영역 종료
    flag[1] = false;
}
```


