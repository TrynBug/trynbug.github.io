---
title: 루프 탈출조건에 atomic bool을 써야하는 이유
date: 2026-02-02 02:14:33 +0900
categories: [멀티스레드, Programming]
tags: [c++, multithread]
description: 루프 탈출조건에 atomic bool을 써야하는 이유
---

## 요약
- 멀티스레드 환경에서 루프를 빠져나오는 조건 변수는 atomic으로 선언해야한다.

## 1. 무한루프에서 빠져나오지 못하는 코드

아래 코드의 1번스레드는 무한루프에서 빠져나오지 못한다.
```cpp
// run 초기값은 true
bool run = true;

// 1번스레드
void Thread1()
{
    // run이 true일 동안 무한루프를 돈다.
    int count = 0;
    while ( run )
    {
        count++;
    }
    std::cout << "thread1 terminated. count=" << count << std::endl;
}

// 2번스레드
void Thread2()
{
    // 3초뒤에 run을 false로 변경한다.
    std::this_thread::sleep_for( std::chrono::seconds( 3 ) );
    run = false;
    std::cout << "thread2 terminated" << std::endl;
}

int main()
{
    // 스레드 실행
    std::thread t1 = std::thread( Thread1 );
    std::thread t2 = std::thread( Thread2 );

    t1.join();
    t2.join();

    Sleep( INFINITE );

    return 0;
}
```

2번 스레드가 3초뒤에 run값을 false로 바꾸는데 왜 1번스레드는 무한루프에서 빠져나오지 못할까?  
어셈블리 코드를 통해 원인을 확인해보았다.  

assembly
```cpp
void Thread1()
{
    int count = 0;
    while ( run )
0x007A1110  cmp         byte ptr [run (07A5060h)],0    // run의 값이 0인지 비교한다.
0x007A1117  je          Thread1+12h (07A1122h)         // 비교결과가 참이면 while루프를 벗어난다.
0x007A1119  nop         dword ptr [eax]  
0x007A1120  jmp         Thread1+10h (07A1120h)         // while루프 내의 코드로 다시 이동한다. (무한루프)
    {
        count++;
    }
    std::cout << "thread1 terminated. count=" << count << std::endl;
0x007A1122  mov         ecx,dword ptr [__imp_std::cout (07A3070h)]  
}
```

어셈블리코드를 확인해보면 컴파일러가 코드를 최적화해버려서 맨 처음에만 run 변수 값을 조사하고,  
그 뒤로는 아예 run 변수값 조사 자체를 하지 않는다.  
그래서 무한루프에 빠진다.  

그러면 이제 run 변수를 atomic<bool> 타입으로 선언했을 때의 코드를 확인해보자.
```cpp
// run을 atomic 으로 선언함
std::atomic<bool> run = true;
```

assembly
```cpp
void Thread1()
{
    while ( run )
0x00791110  mov         cl,byte ptr [run (0795060h)]   // run의 주소에서 값을 가져와 cl 레지스터에 저장한다.
    int count = 0;
0x00791116  xor         eax,eax  
    while ( run )
0x00791118  test        cl,cl                          // cl의 값이 0인지 확인한다
0x0079111A  je          Thread1+1Bh (079112Bh)         // 값이 0이면 while loop를 벗어난다.
0x0079111C  nop         dword ptr [eax]  
0x00791120  mov         cl,byte ptr [run (0795060h)]   // run의 주소에서 값을 가져와 cl 레지스터에 저장한다.
    {
        count++;
0x00791126  inc         eax  
    while ( run )
0x00791127  test        cl,cl                           // cl의 값이 0인지 확인한다.
0x00791129  jne         Thread1+10h (0791120h)          // 값이 0이 아니면 0x00791120 주소 코드로 돌아간다.
    }
    std::cout << "thread1 terminated. count=" << count << std::endl;
0x0079112B  mov         ecx,dword ptr [__imp_std::cout (0793070h)]  
}
```
항상 run 변수의 주소로부터 값을 가져와 사용하기 때문에 run 변수의 값이 변경되면 즉시 감지하여 루프를 빠져나온다.
