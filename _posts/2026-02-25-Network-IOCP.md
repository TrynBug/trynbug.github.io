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
- 일반적인 경우에는 한 시점에 최대로 실행될 수 있는 Worker 스레드의 수는 **CPU 코어 수**가 적당하다고 한다.
1. Worker 스레드의 수
- 만약 처리해야하는 작업중 스레드가 block(다른 I/O 작업을 동기로 요청하거나, 동기화객체(lock)를 기다려야 하는 경우 등)될 가능성이 높다면 Worker 스레드가 많이 필요할 것이다. 왜냐하면 완료통지를 처리하던 스레드가 block 되어있는동안 다른 Worker 스레드가 완료통지를 받아 처리할 것이고, 다른 스레드 또한 block된다면 또다른 스레드가 완료통지를 받아 처리할 것이기 때문에 준비해놓은 Worker 스레드의 수가 부족할 수 있다.
- 반대로 처리해야하는 작업중 스레드가 block될 가능성이 낮다면 Worker 스레드를 적게 유지해도 된다. 왜냐하면 완료통지를 처리하는동안 스레드가 block되지 않는다면 다른 스레드가 실행기회를 얻기 힘들기 때문에 Worker 스레드의 수가 부족할 일은 잘 없을 것이다.


## IOCP 생성하고 사용하기

#### 1. IOCP 생성하기
CreateIoCompletionPort 함수는 IOCP를 생성하는 함수이다.
```cpp
HANDLE WINAPI CreateIoCompletionPort(
  _In_     HANDLE    FileHandle,
  _In_opt_ HANDLE    ExistingCompletionPort,
  _In_     ULONG_PTR CompletionKey,
  _In_     DWORD     NumberOfConcurrentThreads
);
```

IOCP를 생성할 때는 CreateIoCompletionPort 함수를 이렇게 호출해야 한다.
```cpp
// 최대 실행가능한 Worker 스레드 수. 값을 0 입력하면 자동으로 현재 CPU 코어 수로 설정된다.
int numConcurrentThread = 0; 

// IOCP 생성
HANDLE hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, NULL, numConcurrentThread);
```

#### 2. 소켓을 IOCP에 등록하고 receive 시작하기
소켓에 send, receive 요청을 비동기로 한 다음 완료통지를 IOCP로 받으려면 소켓을 IOCP에 등록해야 한다.

```cpp
#include <winsock2.h>
#include <mswsock.h>
#include <iostream>
#include <vector>
#include <thread>

#pragma comment(lib, "ws2_32.lib")

#define BUFFER_SIZE 1024

// 먼저 소켓과 IOCP를 편하게 사용하기 위한 구조체 몇개를 정의한다.
// OVERLAPPED_EX 구조체는 OVERLAPPED 구조체를 확장한 구조체이다. 몇가지 추가 정보가 있다.
enum class IO_TYPE { SEND, RECV };
struct OVERLAPPED_EX
{
  OVERLAPPED overlapped = { 0, };      // OVERLAPPED 구조체
  IO_TYPE ioType = { IO_TYPE::SEND };  // IO 작업이 Send 인지 Receive 인지를 표시
  WSABUF wsaBuf = { 0, };              // 데이터 버퍼
  char buffer[BUFFER_SIZE] = { 0, };   // 데이터 버퍼
};

// Session 구조체는 소켓과 Overlapped 구조체를 모아둔 구조체이다.
struct Session 
{
  SOCKET socket = 0;  // 소켓
  OVERLAPPED_EX send; // 송신용 OVERLAPPED 구조체
  OVERLAPPED_EX recv; // 수신용 OVERLAPPED 구조체
};

// listen socket에서 accept하여 클라이언트의 소켓을 얻었다.
SOCKADDR_IN addr;
ZeroMemory(&addr, sizeof(SOCKADDR_IN));
int lenAddr = sizeof(addr);
SOCKET clientSock = accept(listenSock, (SOCKADDR*)&addr, &lenAddr);

// 클라이언트 소켓으로 Session 객체를 만든다.
Session* pSession = new Session;
pSession->socket = clientSock;

// 소켓을 IOCP에 등록한다. 
CreateIoCompletionPort(
  (HANDLE)clientSock,     // 클라이언트 소켓
  hIOCP,                  // IOCP 핸들
  (ULONG_PTR)pSession,    // CompletionKey. 여기에 입력한 값은 IOCP 완료통지를 받을 때 다시 전달받을 수 있다.
  0                       // 0 입력
);

// 최초의 비동기 receive를 시작한다.
ZeroMemory(&pSession->recv.overlapped, sizeof(OVERLAPPED));
pSession->recv.wsaBuf.len = BUFFER_SIZE;
pSession->recv.wsaBuf.buf = pSession->recv.buffer;
pSession->recv.ioType = IO_TYPE::RECV;  // IO가 receive임을 표시

// WSARecv에 receive용 OVERLAPPED 구조체를 전달하여 비동기IO를 시작한다.
DWORD recvBytes = 0, flags = 0;
WSARecv(pSession->socket, &pSession->recv.wsaBuf, 1, &recvBytes, &flags, &pSession->recv.overlapped, NULL);
```

#### 3. 소켓에 send할일이 있다면 send하기

```cpp
ZeroMemory(&pSession->send.overlapped, sizeof(OVERLAPPED));
pSession->send.wsaBuf.len = sendBytes;              // 보낼 데이터 바이트수
pSession->send.wsaBuf.buf = pSession->send.buffer;  // 보낼 데이터
pSession->send.ioType = IO_TYPE::RECV;              // IO가 send임을 표시

// WSASend에 send용 OVERLAPPED 구조체를 전달하여 비동기IO를 시작한다.
DWORD sendBytes = 0;
WSASend(pSession->socket, &pSession->send.wsaBuf, 1, &sendBytes, 0, &pSession->send.overlapped, NULL);
```

#### 4. Worker 스레드가 IOCP 완료통지를 처리하기

```cpp
// worker 스레드를 담아둘 vector
std::vector<std::thread> workerThreads;

// 원하는 개수만큼 worker 스레드를 만든다.
for(int i=0; i<10; ++i)
{
    workerThreads.emplace_back([hIOCP]() 
    {
        DWORD numByteTrans;
        ULONG_PTR compKey;
        OVERLAPPED_EX* pOverlapped;
        Session* pSession;

        // 무한반복
        while (true) 
        {
            // IOCP로부터 완료통지가 오기를 기다린다. 완료통지가 오면 numByteTrans, compKey, pOverlapped 변수에 값이 들어온다.
            // GetQueuedCompletionStatus 함수를 호출한 스레드는 IOCP에 의해 관리되는 스레드가 된다. 
            BOOL retGQCS = GetQueuedCompletionStatus(hIOCP, &numByteTrans, &compKey, (OVERLAPPED**)&pOverlapped, INFINITE);

            // 오류상황 처리는 생략..

            // CompletionKey 로부터 세션을 얻는다.
            // 소켓을 IOCP와 연결할 때 CreateIoCompletionPort 함수에 CompletionKey로 Session의 포인터를 넣었기 때문이다.
            pSession = (Session*)compKey;

            if (pOverlapped->ioType == IO_TYPE::RECV)
            {
                // 완료통지가 WSARecv에 대한 것이다.
                // numByteTrans 변수에는 받은 바이트 수가 들어온다.
                std::cout << "받은 byte 수: " << numByteTrans << std::endl;

                /*
                  이 부분에는 받은 데이터 처리 로직을 작성한다.
                */ 

                // 받은 데이터를 처리했으므로 다시 receive를 시작한다.
                ZeroMemory(&pSession->recv.overlapped, sizeof(OVERLAPPED));
                pSession->recv.wsaBuf.len = BUFFER_SIZE;
                pSession->recv.wsaBuf.buf = pSession->recv.buffer;
                pSession->recv.ioType = IO_TYPE::RECV;  // IO가 receive임을 표시

                DWORD recvBytes = 0, flags = 0;
                WSARecv(pSession->socket, &pSession->recv.wsaBuf, 1, &recvBytes, &flags, &pSession->recv.overlapped, NULL);
            }
            else if (pOverlapped->ioType == IO_TYPE::SEND)
            {
                // 완료통지가 WSASend에 대한 것이다.
                std::cout << "보낸 byte 수: " << numByteTrans << std::endl;
                /*
                  이 부분에는 데이터 송신이 완료되었을 때의 처리 로직을 작성한다.
                */
            }
        }
    });
}
```

## IOCP의 스레드 관리 방식

