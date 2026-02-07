---
title: Out-of-Order-Execution
date: 2026-02-05 23:13:30 +0900
categories: [CPU, Out-of-Order-Execution]
tags: [cpu, outofordering]
description: Out-of-Order-Execution
---

## 1. Out-of-Order-Execution

참고: STORE = 메모리에 쓰기, LOAD = 메모리에서 읽기

CPU는 성능을 높이기 위해 명령어를 코드에 작성된 순서대로 실행하지 않는다. 이것을 Out-of-Order Execution 이라고 한다.  
예를 들면 STORE 해야 하는 데이터들을 모아두었다가 메모리에 한번에 쓰거나,  
메모리에서 LOAD 해야하는 데이터들을 미리미리 읽어둔다.

STORE 명령어와 데이터를 모아두는 곳을 Store Buffer, LOAD 명령어를 모아두는 곳을 Load Buffer라고 한다.

Store Buffer와 Load Buffer는 서로 통신할 수 있다.  
만약 Load Buffer에서 읽어야 하는 데이터가 Store Buffer에 있다면 Store Buffer에서 가져온다.

![Buffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img.png)


## 2. Store Buffer
데이터를 메모리(캐시메모리, 물리메모리)에 기록하는 작업은 CPU 입장에서 매우 느린 작업이다.  
그래서 CPU는 메모리에 기록이 완료되는것을 기다리는 대신에 store buffer에 넣어두고 다음 명령을 수행한다.  
store buffer에 있는 데이터는 알고리즘에 의해 나중에 메모리에 실제로 write 된다.  

![StoreBuffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img-storebuffer.png)

아래와 같은 코드가 있다.
```cpp
// value_remote는 캐시메모리에 없다. 그래서 물리메모리에서 캐시라인을 가져와야 한다.(작업이 완료되기까지 매우 오래 걸림)
value_remote = 100;

// 다른 변수에 값을 저장하는 코드들
value = 1 + 2;
```
코드만 보면 value_remote에 값이 먼저 저장된 다음 value에 값이 저장되어야 한다.  
그런데 value_remote는 완료되기까지 긴 시간이 걸리기 때문에 CPU는 value_remote = 100; 을 store buffer에만 저장해두고 그 다음 명령을 수행한다.  
그 다음 명령어(value = 1 + 2;)도 계속해서 수행하여 결과를 store buffer에 저장한다.

value_remote의 값이 실제 메모리에 반영되지 않았다고 해도 현재 코어에서는 아무런 문제가 없다.  
왜냐하면 현재 코어에서 value_remote 값이 다시 필요할 경우 store buffer에서 가져다 쓰기 때문이다.

#### Store Buffer의 데이터가 실제로 메모리에 write되는 시점
1. 어떤 명령어 앞의 모든 명령어가 실행 완료 되었을 때
   - 앞의 코드에서 value_remote 와 value 는 일단은 모두 store buffer에 저장되어 있다.
   - 이 때 value = 1 + 2; 의 계산이 완료되었다고 해서 바로 메모리에 반영하는것이 아니라, 앞의 모든 명령어(value_remote = 100; 등)가 완료되어서 메모리에 반영될수있다면 그제서야 value의 결과도 메모리에 반영될 수 있다.
   - 그래서 명령어의 계산이 끝나는 시점은 모두 달라도, 명령어가 실제로 메모리에 write 되는 순서는 코드의 순서와 동일하다.
2. lock 등의 메모리배리어 효과가 있는 명령어를 만났을 때
   - lock 등의 명령어를 수행하려면 lock 이전 명령어가 모두 수행되어야 한다. 
   - 그래서 이런 명령어를 만나면 Store Buffer 내의 데이터를 모두 메모리에 write 한다.
3. 다른 코어와의 데이터 일관성(Cache Coherency)을 맞춰야 할 때
   - 다른 코어가 내 코어가 write 하고있는 데이터를 읽으려고 하면 MESI 프로토콜 등에 의해 Store Buffer 내의 데이터를 메모리에 write 한다.



## 3. Load Buffer
데이터를 메모리(캐시메모리, 물리메모리)에서 읽는 작업은 CPU 입장에서 매우 느린 작업이다.  
그래서 CPU는 메모리에에서 데이터 읽기가 완료되는것을 기다리는 대신에 LOAD 작업들을 Load Buffer에 넣어두고 한번에 관리한다.  

![LoadBuffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img-loadbuffer.png)

아래와 같은 코드가 있다.
```cpp
// value_remote는 캐시메모리에 없다. 그래서 물리메모리에서 캐시라인을 가져와야 한다.(작업이 완료되기까지 매우 오래 걸림)
data1 = value_remote;
val1 = data1 * 10;  // data1 사용

// value_cache는 캐시메모리에 있다. 그래서 빠르게 가져올 수 있다.
data2 = value_cache;
val2 = data2 * 50;  // data2 사용
```

코드만 본다면, val1에 값 입력이 끝난 다음에 val2에 값이 입력되어야 한다.  
여기에서 Load Buffer가 관여한다면 순서가 다음과 같이 될 수 있다:
1. Load Buffer에 value_remote, value_cache 읽기 작업을 관리함
2. value_remote 읽기 시작
3. value_cache 읽기 시작
4. value_cache 읽기 완료됨 -> val2 = data2 * 50; 명령어 수행하고 val2를 store buffer에 저장
5. value_remote 읽기 완료됨 -> val1 = data1 * 10; 명령어 수행하고 val1을 store buffer에 저장

그래서 val2의 계산이 val1보다 먼저 완료될 수 있다.


## 4. x86-64 아키텍처에서의 Out-of-Order Execution

CPU 내부에서는 Store Buffer와 Load Buffer에 의해 명령어가 최적의 순서대로 재정렬되어 실행될 수 있다.  
그런데 메모리에 반영되는 순서는 x86-64 아키텍처에서는 아래 제약을 지켜야 한다.  

#### Store 명령어는 Load 명령어 뒤에 배치될 수 있다(Stores can be reordered after loads).

```cpp
// 아래 2개 STORE 명령어의 메모리 반영 순서는 바꿀 수 없다.
val1 = 1  // STORE
val2 = 2  // STORE
```

```cpp
// 아래 2개 LOAD 명령어의 메모리 반영 순서는 바꿀 수 없다.
a = val1  // LOAD
b = val2  // LOAD
```

```cpp
// 아래 STORE 명령어의 메모리 반영 순서는 LOAD 앞으로 재배치될 수 없다.
a = val1  // LOAD
val2 = 2  // STORE
```

```cpp
// 아래 STORE 명령어의 메모리 반영 순서는 LOAD 다음으로 재배치될 수 있다.
val1 = 1  // STORE
b = val2  // LOAD
```


참고문서:
https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf


## 5. Out-of-Order Execution 관련된 코딩 주의사항

아래 코드의 출력 결과를 예상해보자.
```cpp
int a = 0;
int b = 0;

std::thread t1 = std::thread([&a, &b]()
{
    a = 1;
    std::cout << b;
});

std::thread t2 = std::thread([&a, &b]()
{
    b = 1;
    std::cout << a;
});
```
std::cout 으로 값을 출력하는 시점에서는, a 와 b 중에 적어도 1개는 값이 1일 것이다.  
그래서 출력 결과는 3가지 경우가 있을 거라고 예상할 수 있다.  
`10`, `01`, `11`  
그런데 실제로는 `00` 도 나올 수 있다.  

왜냐하면 Out-of-Order Execution에 의해 아래와 같이 작동할 수 있기 때문이다.
```cpp
std::thread t1 = std::thread([&a, &b]()
{
    a = 1;            // 순서1: a가 코어1의 store buffer에 들어감(코어2는 이 사실을 알 수 없음)
    std::cout << b;   // 순서2: 메모리에서 b를 읽음(b는 0)
});

std::thread t2 = std::thread([&a, &b]()
{
    b = 1;            // 순서3: b가 코어2의 store buffer에 들어감(코어1은 이 사실을 알 수 없음)
    std::cout << a;   // 순서4: 메모리에서 a를 읽음(a는 0)
});
```

예상하지 못한 결과 `00` 이 나오는 것을 방지하려면 Memory Barrier를 사용하면 된다.
```cpp
std::thread t1 = std::thread([&a, &b]()
{
    a = 1;
    MemoryBarrier();  // 메모리 배리어
    std::cout << b;
});

std::thread t2 = std::thread([&a, &b]()
{
    b = 1;
    MemoryBarrier();  // 메모리 배리어
    std::cout << a;
});
```
