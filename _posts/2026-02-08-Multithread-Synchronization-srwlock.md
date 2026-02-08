---
title: SRWLock
date: 2026-02-08 22:01:24 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: SRWLock
---

## 1. SRWLock

SRWLOCK(Slim Reader Writer Lock) 동기화 객체는 Windows에서 사용하는 유저모드 동기화 객체이다.  

SRWLOCK은 쓰기 작업을 위한 배타적 Lock과 읽기 작업을 위한 공유 Lock을 구분하여 사용할 수 있다.  
쓰기 작업을 할 때는 AcquireSRWLockExclusive 함수로 lock을 획득하여 다른 스레드가 접근하지 못하게 한다.  
읽기 작업을 할 때는 AcquireSRWLockShared 함수로 lock을 획득하여 다른 읽기 스레드가 동시에 접근할 수 있도록 한다.  

SRWLOCK은 재귀적으로 획득할 수 없다.  
실수로 2번 연속 lock을 획득하면 데드락에 걸린다.

## 2. SRWLock 사용법

```cpp
class Object
{
public:
    Object()
    {
        InitializeSRWLock(&m_slock);
	}

    ~Object()
    {
		// SRWLOCK은 소멸될때 정리작업을 안해줘도 된다.
	}

public:
    void ExclusiveWork()
    {
        // 쓰기작업을 위해 배타적 lock을 획득한다.
        AcquireSRWLockExclusive(&m_slock);
        
        // do something ...

        // 배타적 lock 해제
        ReleaseSRWLockExclusive(&m_slock);
    }

    void SharedWork()
    {
        // 읽기작업을 위해 공유 Lock을 획득한다.
        AcquireSRWLockShared(&m_slock);

        // do something ...

        // 공유 lock 해제
		ReleaseSRWLockShared(&m_slock);
    }

private:
    SRWLOCK m_slock;
};
```


## 3. SRWLOCK 구조체

SRWLOCK 구조체 내부를 확인해보자.
```cpp
typedef struct _RTL_SRWLOCK {
    PVOID Ptr;
} RTL_SRWLOCK, * PRTL_SRWLOCK;
typedef RTL_SRWLOCK SRWLOCK, *PSRWLOCK;
```

SRWLOCK은 단순히 포인터값 하나만 가지고 Lock을 구현한다.  
포인터값의 각 비트를 플래그처럼 사용하기 때문에 가능하다.  
대략적으로 64bit 값의 하위 4bit는 상태 플래그로 사용하고, 상위 60bit는 주소값으로 사용한다.  

- 0번째 bit : Lock 사용중 여부
- 1번째 bit : Lock을 기다리는 스레드가 있는지 여부
- 2번째 bit : Lock을 해제할 때 기다리는 스레드를 깨우는 작업이 진행중인지 여부
- 3번째 bit : Shared Lock을 사용중인지 여부
- 4~63 bit : 대기중인 스레드를 관리하는 구조체의 주소
