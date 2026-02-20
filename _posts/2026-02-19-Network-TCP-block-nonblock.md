---
title: block 소켓, nonblock 소켓, 비동기 소켓
date: 2026-02-20 11:51:26 +0900
categories: [네트워크, TCP]
tags: [network, TCP]
description: block 소켓, nonblock 소켓, 비동기 소켓
---

## block 소켓과 nonblock 소켓

- block 소켓
  - block 소켓은 현재 작업(send, recv)을 할 수 있으면 작업을 하고, 할 수 없다면 스레드가 block 된다. 
  - 스레드가 block되면 작업을 할 수 있는 상태가 되면 OS가 스레드를 깨워준다.
- nonblock 소켓
  - nonblock 소켓은 현재 작업을 할 수 있으면 작업을 하고, 할 수 없다면 작업을 하지 않는다. 
  - 작업을 하지 않을때는 함수가 SOCKET_ERROR 오류를 리턴하고 시스템 오류코드를 WSAEWOULDBLOCK 로 세팅한다.
  - 그래서 스레드가 block되는 일이 없다.

소켓을 nonblock 소켓으로 설정하는 방법
```cpp
// 소켓 생성
SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);

// nonblock 소켓으로 전환
u_long on = 1;
int retval = ioctlsocket(listen_sock, FIONBIO, &on);
```

## send

send 함수
```cpp
int send(
  SOCKET     s,     // 소켓
  const char *buf,  // 데이터
  int        len,   // 데이터 길이
  int        flags
);
```
send 함수는 TCP 소켓의 송신버퍼에 데이터를 입력한다.  
송신버퍼에 데이터를 입력하는 작업은 thread-safe 하기 때문에 send는 여러 스레드가 동시에 호출해도 된다.  
그런데 send가 thread-safe 하다고 하더라도 여러 스레드에서 동시에 호출하면 send 순서까지는 보장되지 않으므로 가급적 1개 스레드에서 호출하는것이 좋다.  

- block 소켓의 경우
  - send 함수의 len 인자에 입력된 크기만큼의 데이터가 송신버퍼에 입력되기 전까지는 리턴되지 않는다.
  - len 에 1000 을 입력했으면, send 함수의 리턴 값은 반드시 1000 이거나, error 이다.
  - 송신버퍼에 남은 공간이 하나도 없다면 스레드가 block 된다(일반적으로 이런일은 거의 일어나지 않음).
- nonblock 소켓의 경우
  - 송신버퍼에 남은 공간만큼 데이터를 입력한 다음 리턴한다. 그래서 send 함수의 len 인자에 입력된 값보다 작은 값이 리턴될 수 있다.
  - 만약 리턴값이 len보다 작다면 사용자는 나중에 send를 다시 호출하여 못보낸 데이터를 보내야 한다.
  - 송신버퍼에 남은 공간이 하나도 없다면 SOCKET_ERROR 오류를 리턴하고 시스템 오류코드를 WSAEWOULDBLOCK 로 세팅한다. (오류코드는 WSAGetLastError() 함수로 확인)

## recv
```cpp
int recv(
  SOCKET s,     // 소켓
  char   *buf,  // 사용자 수신버퍼
  int    len,   // 수신버퍼 길이
  int    flags
);
```
- block 소켓의 경우
  - 소켓의 수신버퍼에 데이터가 있다면 데이터를 사용자 수신버퍼에 입력하고, 입력한 데이터의 크기를 리턴한다.
  - 소켓의 수신버퍼에 데이터가 없다면 스레드가 block 된다.
- nonblock 소켓의 경우
  - 소켓의 수신버퍼에 데이터가 있다면 데이터를 사용자 수신버퍼에 입력하고, 입력한 데이터의 크기를 리턴한다.
  - 소켓의 수신버퍼에 데이터가 없다면 SOCKET_ERROR 오류를 리턴하고 시스템 오류코드를 WSAEWOULDBLOCK 로 세팅한다. (오류코드는 WSAGetLastError() 함수로 확인)
    - 이 경우에는 잠시 기다렸다가 다시 recv 함수를 호출해야 한다.





## nonblock 소켓은 비동기 소켓인가?

#### 먼저 block 소켓에 recv 하는것과, 파일을 동기로 read 하는것을 비교해보자.

block 소켓에 recv 하는경우:
1. block 소켓에 recv 함수 호출
2. OS에게 소켓 recv 요청을 한다.
3. 소켓 수신버퍼에 데이터가 없으면 OS가 스레드를 block 한다. (데이터가 있으면 즉시 return)
4. NIC 로부터 소켓 수신버퍼에 데이터가 들어왔다는 신호가 오면 OS가 데이터를 사용자버퍼에 전달하고 스레드를 깨운다.

파일을 동기로 read 하는경우:
1. 파일 핸들에 ReadFile 함수 호출
2. OS에게 파일 read 요청을 한다.
3. 캐싱된 파일데이터가 없다면 OS가 스레드를 block 하고 디스크에 I/O 요청을 보낸다. (캐싱된 데이터가 있으면 즉시 return)
4. 디스크에게서 응답이 오면 OS가 데이터를 사용자버퍼에 전달하고 스레드를 깨운다.

둘에 큰 차이가 없으므로 block 소켓은 동기 소켓이라고 볼 수 있다.

#### 이번에는 nonblock 소켓에 recv 하는것과, 파일을 비동기로 read 하는것을 비교해보자.

nonblock 소켓에 recv 하는경우:
1. nonblock 소켓에 recv 함수 호출
2. OS에게 소켓 recv 요청을 한다.
3. 소켓 수신버퍼에 데이터가 없으면 SOCKET_ERROR를 리턴하고 시스템 오류코드를 WSAEWOULDBLOCK 로 세팅한다. (데이터가 있으면 즉시 return)

파일을 비동기로 read 하는경우:
1. 파일 핸들에 ReadFile 함수 호출
2. OS에게 파일 read 요청을 한다.
3. 캐싱된 파일데이터가 없다면 OS가 디스크에 I/O 요청을 보낸다음 false를 리턴하고 시스템 오류코드를 ERROR_IO_PENDING 로 세팅한다. (캐싱된 데이터가 있으면 즉시 return)
4. 나중에 사용자가 Event, APC, IOCP 등을 통해 결과를 받는다.

nonblock 소켓은 지금 작업을 할 수 없으면 작업을 안하는 것이기 때문에 비동기 방식이라고 보기는 어렵다.  
비동기 I/O는 다른 장치에게 작업을 요청해두고 나는 다른일을 하는 것이다.  


## 비동기 소켓
WSASend, WSARecv 함수를 사용하면 소켓을 비동기 방식으로 사용할 수 있다.  
소켓이 block 또는 nonblock으로 설정되어 있는것과 상관없이 비동기 방식으로 사용할 수 있다.  

```cpp
int WSAAPI WSASend(
  [in]  SOCKET                             s,                      // 소켓
  [in]  LPWSABUF                           lpBuffers,              // 보낼 데이터 배열
  [in]  DWORD                              dwBufferCount,          // 보낼 데이터 배열 size
  [out] LPDWORD                            lpNumberOfBytesSent,    // 보낸 데이터 byte
  [in]  DWORD                              dwFlags,                
  [in]  LPWSAOVERLAPPED                    lpOverlapped,           // WSAOVERLAPPED 구조체. 소켓을 비동기 방식으로 사용할려면 여기에 값을 입력해야 한다.
  [in]  LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine     
);

int WSARecv(
  [in]      SOCKET                             s,                       // 소켓
  [in, out] LPWSABUF                           lpBuffers,               // 수신버퍼 배열
  [in]      DWORD                              dwBufferCount,           // 수신버퍼 배열 size
  [out]     LPDWORD                            lpNumberOfBytesRecvd,    // 수신한 데이터 byte
  [in, out] LPDWORD                            lpFlags,
  [in]      LPWSAOVERLAPPED                    lpOverlapped,            // WSAOVERLAPPED 구조체. 소켓을 비동기 방식으로 사용할려면 여기에 값을 입력해야 한다.
  [in]      LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
);
```
WSASend, WSARecv 함수로 비동기로 소켓 작업을 요청하면, 함수는 즉시 리턴되지만 OS가 커널에서 알아서 작업을 해준다.
비동기 소켓 Send, Recv 작업의 완료통지를 받으려면 Event, APC, IOCP 등을 사용할 수 있다.
