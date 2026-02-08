---
title: std::mutex
date: 2026-02-08 22:01:24 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: std::mutex
---

## 1. std::mutex

std::mutex는 커널오브젝트 mutex와 다른 것이다.  
std::mutex는 c++ 표준 유저모드 동기화 객체이다.  

std::mutex는 SRWLock을 사용하여 구현되었다.  
그래서 재귀 획득이 안된다.

std::mutex 사용법
```cpp
class Object
{
public:
    void DoWork()
    {
        // lock을 획득한다.
        std::lock_guard<std::mutex> lock(m_lock);

        // do something ...

        // 함수가 종료될 때 lock_guard의 소멸자가 lock을 자동으로 해제함 
    }

private:
    std::mutex m_lock;
};
```

## 2. std::recursive_mutex
std::recursive_mutex는 재귀 획득이 가능한 c++ 표준 유저모드 동기화 객체이다.  
std::recursive_mutex는 SRWLock을 사용하여 구현되었는데,  
내부적으로 현재 Lock을 소유중인 스레드를 관리하기 때문에 재귀획득이 가능하다.

std::recursive_mutex 사용법
```cpp
class Object
{
public:
    void DoWork()
    {
        // lock을 획득한다.
        std::lock_guard<std::recursive_mutex> lock(m_lock);

        // do something ...

        // 함수가 종료될 때 lock_guard의 소멸자가 lock을 자동으로 해제함 
    }

private:
    std::recursive_mutex m_lock;
};
```

## 3. std::shared_mutex
Read Lock, Write Lock이 가능한 c++ 표준 유저모드 동기화 객체이다.  
SRWLock을 사용하여 구현되었기 때문에 재귀획득이 안된다.  

shared_mutex를 사용할 때 주의할 점:
1. 읽기(Read) 빈도가 쓰기(Write) 빈도보다 압도적으로 높을 때만 shared_mutex를 사용한다.
  - 읽기 빈도와 쓰기 빈도가 비슷하면 std::mutex 보다 성능이 더 안좋다.
2. 읽기 Lock(shared_lock)을 획득한 다음 곧바로 쓰기 Lock(unique_lock)을 획득할 수 없다.
  - 이 상황에서 쓰기 Lock이 필요하다면 읽기 Lock을 해제하고 쓰기 Lock을 획득해야함

std::shared_mutex 사용법
```cpp
class Object
{
public:
    void ExclusiveWork()
    {
        // 쓰기작업을 위해 배타적 lock을 획득한다.
        std::unique_lock<std::shared_mutex> lock(m_sLock);

        // do something ...

        // 함수가 종료될 때 unique_lock의 소멸자가 lock을 자동으로 해제함 
    }

    void SharedWork()
    {
        // 읽기작업을 위해 공유 Lock을 획득한다.
        std::shared_lock<std::shared_mutex> lock(m_sLock);

        // do something ...

        // 함수가 종료될 때 shared_lock의 소멸자가 lock을 자동으로 해제함 
    }

private:
    std::shared_mutex m_sLock;
};
```
