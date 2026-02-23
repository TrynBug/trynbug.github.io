---
title: Overlapped I/O
date: 2026-02-23 00:50:58 +0900
categories: [OS, I/O]
tags: [OS, I/O]
description: Overlapped I/O
---

## Overlapped I/O
Overlapped IO는 비동기 I/O이다.  
비동기 IO는 OS가 I/O작업을 대신 처리해주는 것이다.  
스레드가 OS에게 I/O 작업을 요청해두고, 작업의 완료를 기다리지 않은 채 다른 작업을 계속해서 수행할 수 있다.  
Overlapped IO라는 이름이 붙은 이유는, 스레드가 I/O 작업이 완료된것을 기다리지 않아도 되기 때문에 OS에게 여러개의 IO를 연속해서 요청할 수 있기 때문이다.  
그래서 사실 Overlapped I/O 라는 이름 대신 Asynchronous I/O(비동기 I/O)라고 부르는것이 더 좋다.  

## Overlapped I/O의 완료통지를 받는 방법
Overlapped I/O는 OS가 I/O 작업을 대신 해주는 것이기 때문에 I/O작업을 요청한다음 완료여부를 사용자가 체크해야 한다.  
I/O 작업의 결과값을 완료통지 라고 하며, 완료통지를 받는 방식에는 여러가지가 있다.  

1. Event 객체를 사용하기
  - Overlapped I/O를 요청할 때 OVERLAPPED 구조체에 이벤트객체를 등록해두면 I/O가 완료되었을 때 OS가 이벤트객체를 signal 상태로 바꿔준다.
  - 단점: WSAWaitForMultipleEvents 함수로 이벤트객체가 signal 되었는지를 확인해야 하는데, 이 함수로는 최대 64개(WSA_MAXIMUM_WAIT_EVENTS)의 이벤트객체만 확인할 수 있다. 그래서 소켓이 64개가 넘어가면 사용하기가 힘들어진다.
2. APC Queue
  - Overlapped I/O를 요청할 때 OVERLAPPED 구조체와 CompletionRoutine 함수(I/O가 완료되었을 때 호출할 함수)를 등록해주면 된다.
  - 그러면 내 스레드가 alertable wait 상태가 되었을 때 OS가 완료된 I/O에 대한 CompletionRoutine 함수를 알아서 호출해준다.
  - 단점: 내 스레드를 주기적으로 alertable wait 상태로 만들어 주어야 한다(WaitForSingleObjectEx, SleepEx 함수 등으로). 
  - 단점2 : I/O 완료통지는 I/O를 요청한 스레드의 APC Queue에 입력되기 때문에 I/O를 요청한 스레드만 완료통지를 받을 수 있다. 그래서 요청 스레드와 작업 스레드를 분리할 수가 없다.
3. IOCP
  - 가장 발전된 완료통지 받는 방식. 웬만하면 IOCP를 쓰는게 좋다.

## OVERLAPPED 구조체
OVERLAPPED 구조체는 windows에서 Overlapped I/O를 요청하고 완료통지를 받기위한 핵심 정보이다.  

주의할 점은 OVERLAPPED 구조체는 I/O 작업이 완료되기전까지 절대 메모리에서 해제되면 안된다는 것이다. 그래서 동적할당해서 따로 관리해주어야 한다.  

그리고 같은 대상(예: 1개의 파일핸들, 1개의 소켓)에 여러번의 Overlapped I/O를 요청할 경우 각각의 요청은 서로다른 OVERLAPPED 구조체를 사용해야 한다.  
왜냐하면 OVERLAPPED 구조체는 I/O 작업 1개에 대한 정보이기 때문이다.  
그래서 완료통지도 OVERLAPPED 구조체 단위로 확인해야 한다.  

```cpp
typedef struct _OVERLAPPED {
    ULONG_PTR Internal;
    ULONG_PTR InternalHigh;
    union {
        struct {
            DWORD Offset;
            DWORD OffsetHigh;
        } DUMMYSTRUCTNAME;
        PVOID Pointer;
    } DUMMYUNIONNAME;

    HANDLE  hEvent;
} OVERLAPPED, *LPOVERLAPPED;
```
OVERLAPPED 구조체의 파라미터를 확인해보자.
1. Offset 과 OffsetHigh
  - 파일에 Overlapped I/O를 요청할 때는 작업을 할 위치(offset)을 64bit 값으로 지정해주어야 한다.
    - 그 이유는 파일에 비동기로 여러개의 I/O를 동시에 요청할 경우 파일 포인터 위치가 꼬여서 데이터가 같은위치에 덮어씌워지거나 할 수 있기 때문이다.
  - 소켓에 Overlapped I/O를 요청할 때는 이 값을 0으로 초기화해주어야 한다.
    - 소켓에는 비동기로 여러개의 I/O를 동시에 요청해도 요청이 순서대로 처리되기 때문에 값을 지정하지 않아도 된다. 대신 0으로 초기화해야 한다.

2. hEvent
  - Event 객체로 완료통지를 받고싶을 때 여기에 Event 객체 핸들을 입력한다.

3. Internal
  - Overlapped I/O가 처리되었을 때 오류코드가 여기에 기록된다.

4. InternalHigh
  - Overlapped I/O가 완료되었을 때 실제로 송수신된 바이트 수가 기록된다.

## Overlapped I/O가 동기로 처리되는 경우
Overlapped I/O가 반드시 비동기로 처리되는것은 아니다.  
예를들면 파일에 Read 요청을 비동기로 했는데 요청한 데이터가 시스템 캐시에 로드되어 있는 경우라면 시스템 캐시에 있는 데이터를 동기로 읽는다.  
소켓에 Recv 요청을 비동기로 할 때도 소켓 수신버퍼에 이미 데이터가 있다면 수신버퍼 데이터를 동기로 읽는다.  
소켓에 Send 요청을 비동기로 할 때도 소켓 송신버퍼에 공간이 있다면(사실 거의 대부분의 경우에는 송신버퍼에 공간이 있다.) 송신버퍼에 데이터를 동기로 write한다.  
(I/O가 동기로 처리되어도 Event 객체가 signal 되지 않거나, IOCP Completion Queue에 완료통지가 삽입되지 않거나 하는일은 발생하지 않는다.)  

## Overlapped I/O 처리절차


## Event 객체를 사용하여 완료통지 받기
아래는 Event 객체를 사용하여 소켓에 Overlapped I/O를 요청하는 예제이다.  

```cpp
#pragma comment(lib, "ws2_32")
#include <WinSock2.h>
#include <Windows.h>
#include <iostream>

#define BUFSIZE 500

// 1개 소켓의 정보를 모아놓은 구조체
struct SocketInfo
{
    WSAOVERLAPPED overlapped;   // OVERLAPPED 구조체
    SOCKET sock;                // 소켓
    char buf[BUFSIZE + 1];      // 수신버퍼
    WSABUF wsabuf;              // WSA수신버퍼
};

int nTotalSockets = 0;  // 전체 소켓 수
SocketInfo* SocketInfoArray[WSA_MAXIMUM_WAIT_EVENTS];   // 소켓 정보 저장해두는 전역배열
WSAEVENT EventArray[WSA_MAXIMUM_WAIT_EVENTS];           // 이벤트핸들 저장해두는 전역배열

// accept한 다음 비동기 Recv를 시작하는 함수
void AcceptAndStartRecv()
{
    // liste socket에서 accept하여 소켓을 얻는다.
    SOCKADDR_IN clientaddr;
    int addrlen;
    SOCKET client_sock = accept( listen_sock, (SOCKADDR*)&clientaddr, &addrlen );

    // 소켓정보 구조체를 동적할당하고 초기화한다.
    // Overlapped I/O를 하기 위해서는 I/O가 끝날 때까지 OVERLAPPED 구조체가 반드시 메모리에 존재해야 한다. 그래서 동적할당하여 관리한다.
    SocketInfo* ptr = new SocketInfo;
    ptr->sock = sock;
    ptr->wsabuf.buf = ptr->buf;
    ptr->wsabuf.len = BUFSIZE;

    // Overlapped 구조체 초기화
    ZeroMemory( &ptr->overlapped, sizeof( ptr->overlapped ) );

    // 완료통지를 받을 이벤트객체 생성하고 OVERLAPPED 구조체에 등록한다.
    WSAEVENT hEvent = WSACreateEvent();
    ptr->overlapped.hEvent = hEvent;
    
    // 전역 배열에 SocketInfo 구조체와 이벤트핸들을 저장해둔다.
    SocketInfoArray[nTotalSockets] = ptr;
    EventArray[nTotalSockets] = hEvent;
    ++nTotalSockets;

    // WSARecv 함수로 Overlapped I/O 시작
    DWORD recvbytes = 0;
    DWORD flags = 0;
    DWORD retval = WSARecv( ptr->sock, &ptr->wsabuf, 1, &recvbytes, &flags, &ptr->overlapped, NULL );
    // 리턴값이 SOCKET_ERROR 이고 error code가 WSA_IO_PENDING 이면 Overlapped I/O가 시작된 것이다. 그렇지 않으면 오류.
    if ( retval == SOCKET_ERROR )
    {
        if ( WSAGetLastError() != WSA_IO_PENDING )
        {
            // 오류 발생
            return;
        }
    }
}

// 이벤트 객체를 확인하여 Overlapped I/O가 완료되었는지 체크한다.
// 완료되었으면 받은 데이터를 확인하고 다시 receive를 시작한다.
void CheckOverlappedIO()
{
    // 이벤트가 signal 되기를 기다린다.
    DWORD index = WSAWaitForMultipleEvents( nTotalSockets, EventArray, FALSE, WSA_INFINITE, FALSE );

    // signal 되었으면 몇 번째 이벤트객체인지 인덱스를 얻는다.
    index -= WSA_WAIT_EVENT_0;

    // 이벤트의 signal 상태를 reset
    WSAResetEvent( EventArray[index] );

    // 소켓 정보 얻기
    SocketInfo* ptr = SocketInfoArray[index];

    // Overlapped I/O가 완료되었는지 확인한다.
    DWORD cbTransferred = 0;
    DWORD flags = 0;
    int retval = WSAGetOverlappedResult( ptr->sock, &ptr->overlapped, &cbTransferred, FALSE, &flags );

    // 리턴값이 TRUE가 아니라면 Overlapped I/O가 완료되지 않았거나 오류가 발생한 것이다.
    if ( retval == FALSE || cbTransferred == 0 )
    {
        // Overlapped I/O가 끝나지 않았거나 오류가 발생함
        return;
    }

    // Overlapped I/O가 완료되었으면 ptr->buf 에 데이터가 들어가있다.
    printf( "%s\n", ptr->buf );

    // Overlapped I/O가 완료되었으므로 OVERLAPPED 구조체를 초기화하고 다시 Recv를 시작한다.
    ZeroMemory( &ptr->overlapped, sizeof( ptr->overlapped ) );
    ptr->overlapped.hEvent = EventArray[index];  // 이벤트 핸들 다시 등록
    ptr->wsabuf.buf = ptr->buf;
    ptr->wsabuf.len = BUFSIZE;

    DWORD recvbytes = 0;
    DWORD flags = 0;
    int retval = WSARecv( ptr->sock, &ptr->wsabuf, 1, &recvbytes, &flags, &ptr->overlapped, NULL );  // Recv 다시 시작
    if ( retval == SOCKET_ERROR )
    {
        if ( WSAGetLastError() != WSA_IO_PENDING )
        {
            // 오류 발생
            return;
        }
    }
}
```


## APC Queue를 사용하여 완료통지 받기
아래는 APC Queue를 사용하여 소켓에 Overlapped I/O를 요청하는 예제이다.  

```cpp
#pragma comment(lib, "ws2_32")
#include <WinSock2.h>
#include <Windows.h>
#include <iostream>

#define BUFSIZE 500

// 1개 소켓의 정보를 모아놓은 구조체
struct SocketInfo
{
    WSAOVERLAPPED overlapped;   // OVERLAPPED 구조체
    SOCKET sock;                // 소켓
    char buf[BUFSIZE + 1];      // 수신버퍼
    WSABUF wsabuf;              // WSA수신버퍼
};

// Recv가 완료되었을 때 호출될 CompletionRoutine
void CALLBACK CompletionRoutine( DWORD dwError, DWORD cbTransferred, LPWSAOVERLAPPED lpOverlapped, DWORD dwFlags )
{
    // lpOverlapped 파라미터로 SocketInfo를 얻는다.
    SocketInfo* pInfo = (SocketInfo*)lpOverlapped;

    // 연결 종료 또는 오류 발생여부 체크
    if ( dwError != 0 || cbTransferred == 0 )
    {
        printf( "연결 종료 또는 에러 발생 (Error: %d)\n", dwError );
        closesocket( pInfo->socket );
        delete pInfo;
        return;
    }

    // Overlapped I/O가 성공했으면 buf에 데이터가 들어있다.
    printf( "%s\n", pInfo->buf );

    // Overlapped I/O가 완료되었으므로 OVERLAPPED 구조체를 초기화하고 다시 Recv를 시작한다.
    ZeroMemory( &pInfo->overlapped, sizeof( pInfo->overlapped ) );
    pInfo->wsabuf.buf = pInfo->buf;
    pInfo->wsabuf.len = BUFSIZE;

    // CompletionRoutine 등록하여 Recv 다시 시작
    DWORD recvbytes = 0;
    DWORD flags = 0;
    int retval = WSARecv( pInfo->sock, &pInfo->wsabuf, 1, &recvbytes, &flags, &pInfo->overlapped, CompletionRoutine );  
    if ( retval == SOCKET_ERROR )
    {
        if ( WSAGetLastError() != WSA_IO_PENDING )
        {
            // 오류 발생
            return;
        }
    }
}

// accept한 다음 비동기 Recv를 시작하는 함수
void AcceptAndStartRecv()
{
    // liste socket에서 accept하여 소켓을 얻는다.
    SOCKADDR_IN clientaddr;
    int addrlen;
    SOCKET client_sock = accept( listen_sock, (SOCKADDR*)&clientaddr, &addrlen );

    // 소켓정보 구조체를 동적할당하고 초기화한다.
    // Overlapped I/O를 하기 위해서는 I/O가 끝날 때까지 OVERLAPPED 구조체가 반드시 메모리에 존재해야 한다. 그래서 동적할당하여 관리한다.
    SocketInfo* pInfo = new SocketInfo;
    pInfo->sock = sock;
    pInfo->wsabuf.buf = pInfo->buf;
    pInfo->wsabuf.len = BUFSIZE;

    // Overlapped 구조체 초기화
    ZeroMemory( &pInfo->overlapped, sizeof( pInfo->overlapped ) );

    // WSARecv 함수로 CompletionRoutine 등록하여 Overlapped I/O 시작
    DWORD recvbytes = 0;
    DWORD flags = 0;
    int retval = WSARecv( pInfo->sock, &pInfo->wsabuf, 1, &recvbytes, &flags, &pInfo->overlapped, CompletionRoutine );
    // 리턴값이 SOCKET_ERROR 이고 error code가 WSA_IO_PENDING 이면 Overlapped I/O가 시작된 것이다. 그렇지 않으면 오류.
    if ( retval == SOCKET_ERROR )
    {
        if ( WSAGetLastError() != WSA_IO_PENDING )
        {
            // 오류 발생
            return;
        }
    }
}

// 스레드를 alertable wait 상태로 만들어서 CompletionRountine이 호출되도록 하는 함수
void AlertableWait()
{
    // 스레드를 alertable wait 상태로 만들고 APC 큐에 쌓인 모든 콜백을 실행한다.
    DWORD result = SleepEx( INFINITE, TRUE );

    if ( result == WAIT_IO_COMPLETION )
    {
        // result가 WAIT_IO_COMPLETION이면 APC 콜백이 하나 이상 실행되었음을 의미한다.
    }
    
}
```
