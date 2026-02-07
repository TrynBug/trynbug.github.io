---
title: atomic 변수와 memory order
date: 2026-02-03 11:08:03 +0900
categories: [멀티스레드, Programming]
tags: [multithread, atomic]
description: atomic 변수와 memory order
---

참고: 이 문서는 x86-64 아키텍처 기준입니다.  
ARM 아키텍처에는 해당하지 않습니다.

## 요약



## 1. memory_order

- memory_order_relaxed
  - 가장 느슨한 memory order 옵션. 컴파일러가 작업의 순서를 보장하지 않는다(최적화 가능). 오직 현재 연산의 원자성(Atomic)만 보장한다. 특정 상황에서 성능향상을 기대할 수 있음.
- memory_order_acquire
  - 읽기(Load) 할때만 씀. 현재 Load 이후에 있는 Load/Store 작업은 현재 Load 이전으로 재배치될 수 없다.
- memory_order_release
  - 쓰기(Store) 할때만 씀. 현재 Store 이전에 있는 Load/Store 작업은 현재 Store 이후로 재배치될 수 없다.
- memory_order_acq_rel
  - 읽기+쓰기(Load+Store) 할때만 씀. memory_order_acquire와 memory_order_release를 동시에 적용한다. 
- memory_order_seq_cst
  - 기본값. 가장 강력한 memory_order. 모든 스레드에서 연산이 순서대로 적용되며, 연산이 적용되면 모든 스레드가 해당 값을 볼 수 있다.


## 2. 어셈블리 확인해보기

#### fetch_add
```cpp
    a.fetch_add( 1, std::memory_order_relaxed );
00007FF7056931E8  lock inc    dword ptr [rbp-1]  

    a.fetch_add( 1, std::memory_order_acq_rel );
00007FF7056931EC  lock inc    dword ptr [rbp-1]  

    a.fetch_add( 1, std::memory_order_seq_cst );
00007FF7056931F0  lock inc    dword ptr [rbp-1]  
```
fetch_add는 모두 lock inc 어셈블리 명령어를 사용한다.
fetch_add는 값을 읽은 다음 더해서 쓰는 작업이기 때문에 어쨌든 이것을 원자적으로 하려면 lock이 필요하다.

#### store
```cpp
    a.store( 1, std::memory_order_relaxed );
00007FF7E78031F4  mov         dword ptr [rbp-1],1  

    a.store( 1, std::memory_order_release );
00007FF7E78031FB  mov         dword ptr [rbp-1],1  

    a.store( 1, std::memory_order_seq_cst );
00007FF7E7803202  mov         eax,1  
00007FF7E7803207  xchg        eax,dword ptr [rbp-1]  // a 변수 메모리를 lock 걸고 값을 변경한다.
```
store의 경우에는 std::memory_order_relaxed 와 std::memory_order_release 는 mov 연산인데,  
std::memory_order_seq_cst 는 xchg 이다. xchg는 lock을 걸어야 하기 때문에 속도가 훨씬 느리다.

#### load
```cpp
    int v1 = a.load( std::memory_order_relaxed );
00007FF62868320A  mov         ecx,dword ptr [rbp-1]  

    int v2 = a.load( std::memory_order_release );
00007FF62868320D  mov         eax,dword ptr [rbp-1]  

    int v3 = a.load( std::memory_order_seq_cst );
00007FF628683210  mov         eax,dword ptr [rbp-1]  
```
load는 모두 mov 연산이다.

#### exchange
```cpp
    int v4 = a.exchange( 1, std::memory_order_relaxed );
00007FF7E88A3211  mov         eax,r12d  
00007FF7E88A3214  xchg        eax,dword ptr [rbp-11h]  

    int v5 = a.exchange( 1, std::memory_order_acq_rel );
00007FF7E88A3217  mov         ecx,r12d  
00007FF7E88A321A  xchg        ecx,dword ptr [rbp-11h]  

    int v6 = a.exchange( 1, std::memory_order_seq_cst );
00007FF7E88A321D  mov         eax,r12d  
00007FF7E88A3220  xchg        eax,dword ptr [rbp-11h]  
```
exchange는 모두 xchg 연산이다. lock을 사용한다.  
exchange는 현재값을 가져온 다음 store해야 하기 때문에 lock이 필요하다.

#### 결론
atomic을 쓸 때 속도 향상을 원한다면 먼저 store 또는 exchange할 때 std::memory_order_relaxed를 사용할 수 있는지 고려하는것이 좋다.
나머지 경우는 속도에 큰 차이가 있지는 않다.


## 3. 컴파일러 최적화
생성되는 어셈블리 코드가 같다고 해도, 컴파일러 최적화 관점에서 보면 차이가 있을 수 있다.  
std::memory_order_relaxed 는 순서를 보장해도 되지 않기 때문에 컴파일러가 최적화 과정에서 코드의 위치를 재배치할 수 있다.  
그런데 큰 차이가 있다고 보기는 힘들다.

아래 2개 함수를 비교해보자.
```cpp
int g_data = 0;
std::atomic<int> g_v;

void test_relaxed( std::atomic<int>& a )
{
    g_data = 1;
    int temp = a.load( std::memory_order_relaxed );  // memory_order_relaxed 사용
    g_data = 2;

    g_v.store( g_data * temp, std::memory_order_relaxed );
}

void test_seq_cst( std::atomic<int>& a )
{
    g_data = 1;
    int temp = a.load( std::memory_order_seq_cst );  // memory_order_seq_cst 사용
    g_data = 2;

    g_v.store( g_data * temp, std::memory_order_relaxed );
}
```

둘의 어셈블리를 보면,
```cpp
// test_relaxed 함수
void test_relaxed( std::atomic<int>& a )
{
    g_data = 1;   // g_data = 1; 코드는 컴파일러가 불필요한 코드로 보고 삭제하였다.
    int temp = a.load( std::memory_order_relaxed );
00007FF6C46F3138  mov         eax,dword ptr [rcx]  
    g_data = 2;

    g_v.store( g_data * temp, std::memory_order_relaxed );
00007FF6C46F313A  add         eax,eax  
00007FF6C46F313C  mov         dword ptr [g_data (07FF6C4726B54h)],2  
00007FF6C46F3146  mov         dword ptr [g_v (07FF6C4726B58h)],eax  
}
00007FF6C46F314C  ret  

// test_seq_cst 함수
void test_seq_cst( std::atomic<int>& a )
{
    int temp = a.load( std::memory_order_seq_cst );
00007FF6C46F3150  mov         eax,dword ptr [rcx]  
    g_data = 1;
00007FF6C46F3152  mov         dword ptr [g_data (07FF6C4726B54h)],1  // 여기서는 g_data = 1; 코드가 삭제되지 않았다.
    g_data = 2;
00007FF6C46F315C  add         eax,eax  
00007FF6C46F315E  mov         dword ptr [g_data (07FF6C4726B54h)],2  

    g_v.store( g_data * temp, std::memory_order_relaxed );
00007FF6C46F3168  mov         dword ptr [g_v (07FF6C4726B58h)],eax  
}
00007FF6C46F316E  ret  
```

컴파일러가 코드를 약간 최적화해줄 수 있으나 큰 차이는 없어보인다.
