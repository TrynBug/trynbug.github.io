---
title: WaitOnAddress 동기화방식
date: 2026-02-08 15:53:27 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: WaitOnAddress 동기화방식
---

## 1. WaitOnAddress 동기화

Windows의 WaitOnAddress 동기화 기능은 linux 시스템의 Futex(Fast Userspace Mutex)에 대응된다.  

WaitOnAddress 동기화는 "특정 메모리 주소의 값이 내가 원하는 값으로 바뀔 때 운영체제가 내 스레드를 깨워주는 방식" 이다.  

이 방식이 가지는 장점은 다음과 같다:
1. 스레드를 block 하기위해 Semaphore와 같은 커널오브젝트를 사용하지 않는다. 단지 메모리주소 하나만이 필요하기 때문에 메모리가 크게 절약된다.
2. 내가 Lock을 해제하여 다른 스레드를 깨워도, 다른 스레드가 완전히 깨어나기 전에 내가 곧바로 다시 Lock을 획득하면 내가 Lock을 획득할 수 있다.
   - 이렇게 되면 다른 스레드가 완전히 깨어나기 전에 내가 또 작업을 할 수 있기 때문에 CPU가 매우 효율적으로 사용된다.

일단 사용법을 알아보자.  
```cpp
class Object
{
private:
    long m_lock = 0;  // Lock에 사용할 변수

public:
    // Lock 획득 시도
    void AcquireLock()
    {
        // m_lock이 TRUE이면 다른 스레드가 사용중, 그렇지 않으면 사용자가 없는것이다.
        long lCompare = TRUE;
        while (TRUE == InterlockedExchange(&m_lock, TRUE))  // 내가 m_lock을 FALSE 에서 TRUE 로 변경하였다면 return(lock 획득), 그렇지않으면 WaitOnAddress
        {
            // m_lock이 TRUE 이었기 때문에 다른 스레드가 사용중인 것이다. 내 스레드를 block 한다.
            WaitOnAddress(&m_lock, &lCompare, sizeof(m_lock), INFINITE);

            /* 이 위치에 도달했다면 2가지 상황 중의 하나이다.
            1. 내 스레드가 WaitOnAddress 함수로 block 되었었는데, 다른 스레드가 WakeByAddressSingle 함수를 호출하여 깨어났음. 정상적인 상황.
            2. 다른 스레드가 ReleaseLock 함수를 호출하였는데, 함수 내부에서 InterlockedExchange(&m_lock, FALSE); 한 직후에 내가 WaitOnAddress 를 호출한 상황.
               그러니까 m_lock이 FALSE가 되자마자 내가 WaitOnAddress를 호출해서 WaitOnAddress 함수가 곧바로 return된 것이다.
               이 상황에서는 내 스레드, 그리고 WakeByAddressSingle 함수로 깨어난 스레드 총 2개의 스레드가 동시에 임계영역에 진입할 수 있다.
               2번째 상황에서 2개의 스레드가 동시에 진입하는것을 막기 위해 while (TRUE == InterlockedExchange(&m_lock, TRUE)) 를 통해 한번 더 검사하여 둘 중 1개의 스레드만 진입할 수 있도록 한다.
            */
        }
    }

    // Lock 해제
    void ReleaseLock()
    {
        // m_lock을 FALSE로 바꾸어 lock이 해제되었음을 표시한다.
        InterlockedExchange(&m_lock, FALSE);
        // m_lock 값이 FALSE로 바뀌기를 기다리고 있던 다른 스레드를 깨운다.
        WakeByAddressSingle(&m_lock);
    }

    // Lock 걸고 작업하는 함수
    void DoWork()
    {
        // Lock 획득
        AcquireLock();

        // Do Something... 
        
        // Lock 해제
        ReleaseLock();
    }
};
```

#### WaitOnAddress 함수
```cpp
BOOL WaitOnAddress(
  [in]           volatile VOID *Address,           // Address 주소를 대상으로 대기.
  [in]           PVOID         CompareAddress,     // *CompareAddress 값과 *Address 값이 다르면 즉시 return, 같으면 block
  [in]           SIZE_T        AddressSize,        // *Address 값 size
  [in, optional] DWORD         dwMilliseconds      // timeout
);
```
WaitOnAddress 함수를 호출하여 스레드가 block 되었다면,  
깨어나기 위해서는 동일 프로세스내의 다른 스레드가 *Address 값이 변경되었다는 것을 알리는 WakeByAddressSingle 또는 WakeByAddressAll 함수를 호출해야 한다.  
그래서 WakeByAddressSingle 함수를 호출하는 스레드가 함수를 호출하기 전 *Address 값을 먼저 변경해야 한다.  

WaitOnAddress 함수는 성공하였다면 TRUE를 리턴하고, 오류가 발생하였다면 FALSE를 리턴한다.  

WaitOnAddress 함수는 while loop 내에서 Sleep을 사용하는 것보다 효과적인데 스케줄러를 방해하지 않기 때문이다.  
또한 event 오브젝트를 사용하는것 보다 간단한데, event 오브젝트를 생성하고 초기화할 필요가 없는데다 동기화를 신경쓸 필요가 없기 때문이다.  

WaitOnAddress 함수는 Address가 signaled 되었을 때 리턴되는것을 보장하지만, 다른 이유로도 리턴될 수 있다.  
때문에 WaitOnAddress가 리턴된 뒤에는 반드시 *Address 값이 *CompareAddress 값과 다른지를 한번 더 확인해야 한다.  
WaitOnAddress 함수가 리턴될 수 있는 다른 조건은 다음과 같다:  
- 낮은 메모리 상태
- A previous wake on the same address was abandoned(?)
- Executing code on a checked build of the operating system(?)

유저영역 메모리에 Address 별로 현재 대기 중인 스레드 ID를 관리하는 테이블이 있다.  
이 테이블은 Address 값을 key로 하는 hash table이다.  

WakeByAddressSingle 함수를 호출하면 기다리는 스레드가 있을 경우 커널모드로 진입하여 하나의 스레드를 깨우고, 기다리는 스레드가 없을 경우 즉시 리턴된다.  



## 2. WaitOnAddress 동기화 방식이 CPU를 효율적으로 쓰는 방식
Lock을 획득하기위한 AcquireLock() 함수를 보면, WaitOnAddress 함수가 return 되었어도 m_lock의 값을 한번 더 체크한다.  
왜냐하면 Lock이 해제되었을 대기중인 스레드와 내 스레드가 또다시 Lock 획득을 경쟁할 수 있기 때문이다.  
여러개의 스레드가 동시에 임계영역에 진입하는 것을 막기 위해 m_lock의 값을 한번 더 체크한다.  

이런 특징 때문에 스레드가 더 효율적으로 돌 수 있다.  
다른 스레드가 완전히 깨어나기 전에 내가 또 Lock을 획득하여 작업을 할 수 있기 때문이다.  

아래 코드를 확인해보자.
```cpp
// CRITICAL_SECTION은 내부적으로 WaitOnAddress 방식을 사용하기 때문에 CRITICAL_SECTION으로 테스트 하였다.
int main()
{
    CRITICAL_SECTION cs;
    InitializeCriticalSection(&cs);

    // 스레드 함수
    auto proc = [&cs]()
    {
        while (true)
        {
            // Lock을 획득하고, Lock 획득에 걸린시간을 측정하였다.
            std::chrono::time_point<std::chrono::steady_clock> start = std::chrono::steady_clock::now();
            EnterCriticalSection(&cs);
            std::chrono::time_point<std::chrono::steady_clock> end = std::chrono::steady_clock::now();

            // 출력
            std::cout << "스레드: " << std::this_thread::get_id() << ", Lock 획득 대기 시간: " << 
                std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << " 밀리초" << std::endl;
            
            // Lock 해제
            LeaveCriticalSection(&cs);
        }
    };

    // 2개 스레드 시작
    std::thread t1(proc);
    std::thread t2(proc);

    t1.join();
    t2.join();

    return 0;
}
```

출력 결과를 예상해보면, 2개 스레드가 Lock 획득과 해제를 무한으로 반복하고 있기 때문에 2개 스레드가 번갈아가면서 실행될 것처럼 보인다.  
하지만 실제 출력 결과는 다음과 같다.  
![Output](/assets/img/posts/2026-02-08-Multithread-Synchronization_WaitOnAddress-img.png)

  
  
1개 스레드가 일정시간 작업을 계속해서 하다가, 다른 1개 스레드가 일정시간 작업을 계속하고, 다시 다른 스레드가 일정시간 작업을 계속하는 식으로 작동하고 있다.  
이렇게 되는 이유는 1개 스레드가 Lock을 해제한다음 다시 Lock을 획득하려고 할 때 다른 스레드가 아직 깨어나지 않았다면 자신이 Lock을 획득하기 때문이다.  
다른 스레드가 깨어날 때까지 기다릴 필요가 없기 때문에 CPU를 효율적으로 사용하게 된다.
