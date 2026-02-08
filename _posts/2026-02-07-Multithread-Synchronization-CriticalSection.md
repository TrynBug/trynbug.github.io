---
title: CriticalSection 동기화객체
date: 2026-02-07 20:15:27 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: CriticalSection 동기화객체
---

## 1. CriticalSection 동기화 객체

CriticalSection 동기화 객체는 Windows에서 사용하는 유저모드 동기화 객체이다.  
현재 스레드가 임계영역에 진입이 가능한지를 유저영역에서 판단한다.  
그래서 다른 스레드가 임계 영역에 들어가 있지 않는 이상 임계 영역을 굉장히 빠른 속도로(뮤텍스를 얻는 것보다 훨씬 더 빠르다) 얻을 수 있다.  
뮤텍스(std::mutex가 아닌 커널오브젝트 뮤텍스)는 현재 스레드가 임계영역에 진입이 가능한지를 판단하는것도 커널모드에서 수행해야 한다.  
그래서 뮤텍스는 속도가 훨씬 더 느리다.  

CriticalSection이 유저모드 동기화 객체라고 해도 커널모드를 아예 사용하지 않는것은 아니다.  
현재 스레드가 임계영역에 진입할 수 없다면 스레드를 block 해야 하는데, 이것은 커널의 기능이기 때문이다. 이 때는 커널모드로 전환해야 한다.  

CriticalSection은 같은 스레드에 의해 재귀적으로 계속해서 획득될 수 있다.  
그러니까 EnterCriticalSection 함수를 연속적으로 여러번 호출해도 괜찮다는 의미이다.  
CRITICAL_SECTION 구조체에는 OwningThread 변수가 있는데, 만약 EnterCriticalSection 함수를 호출한 스레드의 ID가 OwningThread 에 저장된 스레드 ID와 동일하다면 중복 획득이라도 허용한다.  
SRWLock 등은 재귀적으로 획득을 허용하지 않기 때문에 획득을 연속으로 하면 deadlock에 걸린다.  

재귀적 획득을 허용하는 것은 함수 설계에 상당한 이점이 있다.  
보통 함수 하나 전체를 스레드에 안전한 함수로 만드는데, 이렇게 하기 위해 함수 시작 부분에서 EnterCriticalSection 함수를 호출해야 한다.  
이러한 함수들이 여러개 있고, 이런 함수들을 이것저것 갖다쓰면 동일한 CriticalSection을 여러번 획득하는 경우가 발생할 수 있다.  
이렇게 해도 문제가 발생하지 않는다.  


## 2. CRITICAL_SECTION 사용법

```cpp
class Object
{
public:
    Object()
    {
        // 객체를 생성할 때 CRITICAL_SECTION을 초기화한다.
        InitializeCriticalSection(&m_cs);
    }

    ~Object()
    {
        // 객체가 소멸될 때 CRITICAL_SECTION을 제거한다.
        DeleteCriticalSection(&m_cs);
    }

    void Execute()
    {
        // CRITICAL_SECTION 획득
        EnterCriticalSection(&m_cs);

        // 임계영역
        m_value++;

        // CRITICAL_SECTION 해제
        LeaveCriticalSection(&m_cs);
    }

private:
    CRITICAL_SECTION m_cs;

    int m_value = 0;
};
```


## 3. CRITICAL_SECTION 구조체
CRITICAL_SECTION 구조체가 어떻게 되어있는지 알아보자.

```cpp
typedef struct _RTL_CRITICAL_SECTION {
    PRTL_CRITICAL_SECTION_DEBUG DebugInfo;  // 시스템이 디버깅을 위해 사용하는 정보

    //
    //  The following three fields control entering and exiting the critical
    //  section for the resource
    //

    LONG LockCount;             // Lock 횟수. 아무도 사용하고 있지 않다면 값은 -1이다. -1이 아니라면 어떤 스레드가 소유중인 것이다.
    LONG RecursionCount;        // 재귀 획득 횟수. 초기값은 0이다. 동일 스레드가 Lock을 획득할 때마다 1씩 증가한다.
    HANDLE OwningThread;        // Lock을 획득한 스레드 핸들
    HANDLE LockSemaphore;       // 스레드를 block할 때 사용하는 Semaphore 커널오브젝트의 핸들
    ULONG_PTR SpinCount;        // Lock을 다른 스레드가 사용중일 경우 spin을 돌며 기다릴 횟수
} RTL_CRITICAL_SECTION, * PRTL_CRITICAL_SECTION;
typedef RTL_CRITICAL_SECTION CRITICAL_SECTION;
```

- LockCount
  - 현재 Lock을 획득한 스레드가 있는지 확인하는데 사용하는 값이다. 
  - -1이 아니라면 다른 스레드가 소유중인 것이다.
- OwingThread
  - Lock을 획득한 스레드 핸들. Lock 획득을 시도했을 때 재귀 획득인지 판단하는데 사용된다.
- LockSemaphore
  - Lock을 획득하려 했는데 획득할 수 없다면 스레드가 block 되어야 한다.
  - 이 때 Semaphore 커널오브젝트를 생성하여 다른 스레드가 Lock을 해제할 때까지 이 오브젝트를 기다리도록 한다.
- SpinCount
  - Lock을 획득하려 하는데 다른 스레드가 이미 획득중일 경우, 곧바로 block되지 않고 SpinCount 만큼 루프를 돌며 대기하다가 그래도 Lock을 획득하지 못하면 그때 스레드가 block 된다.
  - 기본값은 2000 정도이다.


## 4. Lock 획득을 시도했는데 획득하지 못했을 때의 동작 분석
Lock 획득을 시도했는데 획득하지 못했을 때 어떤 로직들이 수행되는지 어셈블리 코드로 분석해보았다.

```cpp
        // Lock 획득 시도
        EnterCriticalSection(&cs);
00007FF7E8401010  lea         rcx,[cs (07FF7E8403628h)]                                // rcx에 cs 주소를 넣음
00007FF7E8401017  call        qword ptr [__imp_EnterCriticalSection (07FF7E8402000h)]  // __imp_EnterCriticalSection(&cs) 함수 호출

        // __imp_EnterCriticalSection(&cs) 함수 내부
00007FFF9109FAA0  sub         rsp,28h
00007FFF9109FAA4  mov         rax,qword ptr gs:[30h]        // rax에 gs:[30h] 주소를 저장한다. gs 위치에는 대부분 현재 스레드의 어떤 정보가 저장되어 있다.
00007FFF9109FAAD  lock btr    dword ptr [rcx+8],0           // cs.LockCount의 0번째 bit값을 CF flag에 저장하고, 0번째 bit값을 0으로 바꾼다. 이것을 원자적으로 수행한다.
                                                            // lock btr 에서 btr은 Bit Test and Reset 을 말한다. 두 번째 피연산자(0)는 bit 위치를 말한다. 
                                                            // 여기의 btr 연산자는 첫 번째 피연산자([rcx+8])의 0번째 bit를 CF flag에 저장하고, 0번째 bit를 0으로 reset한다.
                                                            // lock btr은 btr 연산을 원자적으로 수행한다.
                                                            // 그러니까 현재 cs.LockCount 의 값을 CF flag에 저장해두고, cs.LockCount 값을 0으로 바꾼다. 만약 바꾸기 전 값이 1이었다면 lock을 획득한다. 바꾸기 전 값이 0이었으면 lock을 획득하지 못한다.

00007FFF9109FAB3  mov         rax,qword ptr [rax+48h]                               // rax+48h 위치의 값을 rax에 넣는다. 이 값은 스레드 ID인듯하다.
00007FFF9109FAB7  jae         RtlEnterCriticalSection+2Ch (07FFF9109FACCh)          // CF flag가 1일 경우 lock을 획득한 것이다. 
                                                                                    // CF flag가 0일 경우(Lock 획득 불가) 07FFF9109FACC 주소 코드로 이동한다.

    // 여기는 Lock 획득이 가능하다고 판단되었을 때이다.
00007FFF9109FAB9  mov         qword ptr [rcx+10h],rax        // cs.OwningThread 에 스레드 ID를 넣는다.
00007FFF9109FABD  xor         eax,eax                        
00007FFF9109FABF  mov         dword ptr [rcx+0Ch],1          // cs.RecursionCount 에 1 입력
00007FFF9109FAC6  add         rsp,28h                        
00007FFF9109FACA  ret                                        // return. Lock 획득
00007FFF9109FACB  int         3                              

    // 여기는 Lock 획득이 불가능하다고 판단되었을 때이다. 
00007FFF9109FACC  cmp         qword ptr [rcx+10h],rax        // 현재 스레드 ID를 cs.OwningThread 와 비교한다.
00007FFF9109FAD0  jne         RtlEnterCriticalSection+3Dh (07FFF9109FADDh)       // 같지 않다면 07FFF9109FADD 주소 코드로 이동한다.
00007FFF9109FAD2  inc         dword ptr [rcx+0Ch]            
00007FFF9109FAD5  xor         eax,eax                        
00007FFF9109FAD7  add         rsp,28h                        
00007FFF9109FADB  ret                      // return. lock을 획득하지 못했는데 스레드ID를 확인해보니 재귀적으로 lock을 획득한거라서 Lock 획득
00007FFF9109FADC  int         3  

    // Lock을 획득하지 못함
00007FFF9109FADD  call        RtlpEnterCriticalSectionContended (07FFF9109FAF0h)    // RtlpEnterCriticalSectionContended 함수 호출 
```

계속해서 RtlpEnterCriticalSectionContended 함수 내부
```
// 일련의 함수들을 호출
call RtlpEnterCriticalSectionContended
spin lock으로 잠시 대기...
call RtlpWaitOnCriticalSection
call RtlpCreateDeferredCriticalSectionEvent
call RtlpAddDebugInfoToCriticalSection
call RtlpWaitOnAddress
...
syscall    // syscall 이 나타나면 커널영역으로 진입하는 것이다. 스레드를 block하기위해 커널영역에 진입한다. 여기서부터는 어셈블리 코드를 볼 수 없다. 
스레드가 block 됨
```

스레드가 block에서 깨어남
```cpp
// 호출된 함수들 return
return RtlpWaitOnAddress
return RtlpAddDebugInfoToCriticalSection
return RtlpCreateDeferredCriticalSectionEvent
return RtlpWaitOnCriticalSection
return RtlpEnterCriticalSectionContended

// 일련의 작업을 마치고 __imp_EnterCriticalSection 함수에서 return 하여 lock을 획득한다.
return __imp_EnterCriticalSection
```

임계영역에 진입하여 작업을 마치고, 이제 Lock을 해제한다.
```cpp
    // Lock 해제 시작
    LeaveCriticalSection(&cs);
00007FF76CE41161  lea         rcx,[cs (07FF76CE44620h)]  
00007FF76CE41168  call        qword ptr [__imp_LeaveCriticalSection (07FF76CE42008h)]  // __imp_LeaveCriticalSection 함수 호출

// __imp_LeaveCriticalSection 함수 내부
ntdll.dll!RtlLeaveCriticalSection(void):
00007FFF9109F230  mov         qword ptr [rsp+18h],rbx  
00007FFF9109F235  mov         qword ptr [rsp+20h],rsi  
00007FFF9109F23A  push        rdi  
00007FFF9109F23B  sub         rsp,20h  
00007FFF9109F23F  sub         dword ptr [rcx+0Ch],1               // cs.RecursionCount 값을 1 감소시킨다.
00007FFF9109F243  mov         rbx,rcx  
00007FFF9109F246  jne         RtlLeaveCriticalSection+33h (07FFF9109F263h)  
00007FFF9109F248  mov         qword ptr [rcx+10h],0               // cs.OwningThread 에 0을 입력한다.
00007FFF9109F250  mov         eax,0FFFFFFFEh  
00007FFF9109F255  mov         ecx,0FFFFFFFFh  
00007FFF9109F25A  lock cmpxchg dword ptr [rbx+8],ecx  
                  // cmpxchg는 Compare and Exchange 이다. AL, AX, EAX, RAX 레지스터 값을 첫번째 피연산자(destination operand)와 비교한다.
                  // 만약 두 값이 같다면, 두 번째 피연산자(source operand) 값이 destination operand에 입력된다. 그리고 ZF flag가 세팅된다.
                  // 같지 않으면 destination operand 값이 AL, AX, EAX, RAX 레지스터에 입력된다.
                  // CF, PF, AF, SF, OF flag는 비교 결과에 따라 세팅된다.
                  // 64bit 플랫폼에서는 RAX 레지스터만이 사용 가능하다.
                  // lock cmpxchg는 cmpxchg를 원자적으로 수행한다.
                  // 현재 rax 에는 0FFFFFFFEh 가 입력되어 있다. 이 값은 -2이다. ecx는 -1 이다. rbx+8 은 cs.LockCount 이다. 
                  // cs.LockCount 값은 LeaveCriticalSection 함수 호출 시점에 CriticalSection을 기다리는 스레드가 없을 경우 -2 이었고, 
                  // 기다리는 스레드가 1개 있을 경우 -6, 기다리는 스레드가 2개일 경우 -10, ... 이었다.
                  // cs.LockCount 값이 -2 라면 cs.LockCount에 -1을 입력하고, 그렇지 않으면 rax에 cs.LockCount 를 입력한다.
00007FFF9109F25F  mov         esi,eax  
00007FFF9109F261  jne         RtlLeaveCriticalSection+46h (07FFF9109F276h)     // ZF flag가 0일 경우(cs.LockCount 값이 -2가 아닐경우, 즉 기다리는 다른 스레드가 있을 경우) jump
00007FFF9109F263  mov         rbx,qword ptr [rsp+40h]  
00007FFF9109F268  xor         eax,eax  
00007FFF9109F26A  mov         rsi,qword ptr [rsp+48h]  
00007FFF9109F26F  add         rsp,20h  
00007FFF9109F273  pop         rdi  
00007FFF9109F274  ret                      // Lock 획득을 기다리는 다른 스레드가 없다고 판단되면 여기서 return 한다.
00007FFF9109F275  int         3  

// Lock 획득을 기다리는 다른 스레드가 있다면 커널모드로 진입 후 기다리는 스레드를 꺠운다.
00007FFF9109F276  test        byte ptr [rbx+8],1 
00007FFF9109F27A  jne         memset+0F088h (07FFF91122E88h)  
00007FFF9109F280  mov         r9,qword ptr [rbx+18h]  
00007FFF9109F284  test        r9,r9  
00007FFF9109F287  je          RtlLeaveCriticalSection+128h (07FFF9109F358h)  
// 아래는 생략함...
return __imp_LeaveCriticalSection  // return
```
