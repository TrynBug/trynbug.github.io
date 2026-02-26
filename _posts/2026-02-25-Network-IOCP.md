---
title: IOCP
date: 2026-02-25 18:42:40 +0900
categories: [네트워크, IOCP]
tags: [network, IOCP]
description: IOCP
---

## IOCP
IOCP(I/O Completion Port)는 비동기 I/O(Overlapped I/O) 작업의 완료통지를 받는 방식이다.  
소켓에 receive 또는 send 요청을 비동기로 해 놓고, I/O가 완료되면 사용자가 원하는 시점에 완료통지를 받을 수 있다.  

IOCP를 사용하기 전에 먼저 2가지를 결정해야 한다.
1. 한 시점에 최대로 실행될 수 있는 Worker 스레드의 수
- Worker 스레드는 IOCP의 완료통지를 받아서 작업을 처리하는 스레드이다.
- CPU 코어 수가 4개이고, Worker 스레드의 수가 16개 일때를 생각해보자. 코어가 4개이기 때문에 한 시점에 동시에 돌 수 있는 스레드는 최대 4개이다. 
  - IOCP에 완료통지가 16개가 도착하여 Worker 스레드 16개가 동시에 완료통지를 처리한다고 생각해보자. 그러면 스레드들 간의 context switching이 빈번하게 발생하여 프로그램 성능이 떨어지게 된다.
  - 다른경우로, IOCP에 완료통지가 16개가 도착하여도 Worker 스레드가 최대 4개까지만 동시에 실행될수있다고 제한을 두었다고 하자. 그러면 4개 스레드가 완료통지를 4개씩 처리할 것이고, context switching은 거의 발생하지 않을 것이다. 그래서 프로그램의 성능이 향상된다.
- 일반적인 경우에는 한 시점에 최대로 실행될 수 있는 Worker 스레드의 수는 **CPU 코어 수 * 2**가 적당하다고 한다.
2. Worker 스레드의 수
- 만약 처리해야하는 작업중 스레드가 block(다른 I/O 작업을 동기로 요청하거나, 동기화객체(lock)를 기다려야 하는 경우 등)될 가능성이 높다면 Worker 스레드가 많이 필요할 것이다. 왜냐하면 완료통지를 처리하던 스레드가 block 되어있는동안 다른 Worker 스레드가 완료통지를 받아 처리할 것이고, 다른 스레드 또한 block된다면 또다른 스레드가 완료통지를 받아 처리할 것이기 때문에 준비해놓은 Worker 스레드의 수가 부족할 수 있다.
- 반대로 처리해야하는 작업중 스레드가 block될 가능성이 낮다면 Worker 스레드를 적게 유지해도 된다. 왜냐하면 완료통지를 처리하는동안 스레드가 block되지 않는다면 다른 스레드가 실행기회를 얻기 힘들기 때문에 Worker 스레드의 수가 부족할 일은 잘 없을 것이다.


## IOCP 생성하기

```cpp
HANDLE WINAPI CreateIoCompletionPort(
  _In_     HANDLE    FileHandle,
  _In_opt_ HANDLE    ExistingCompletionPort,
  _In_     ULONG_PTR CompletionKey,
  _In_     DWORD     NumberOfConcurrentThreads
);
```
