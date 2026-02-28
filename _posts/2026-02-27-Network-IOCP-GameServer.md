---
title: IOCP와 게임서버
date: 2026-02-27 22:59:17 +0900
categories: [네트워크, IOCP]
tags: [network, IOCP]
description: 게임서버에서 IOCP를 사용할 때 고려사항
---

## 1. 소켓에 WSARecv 할때의 수신버퍼는 링버퍼(ringbuffer)를 사용한다.
링버퍼가 아닌 일반적인 배열 버퍼를 사용자 수신버퍼로 사용한다고 생각해보자.  
```cpp
char* buffer[1000];  // 배열 버퍼
```
소켓에 WSARecv 함수를 호출하였고, 100bytes의 데이터를 수신받았다.  
그런데 TCP 특성상 쪼개진 패킷 데이터를 수신할 수 있다.  
예를들면 100bytes중에 앞의 80bytes는 정상적인 1개 패킷의 데이터이고, 나머지 20bytes는 다른 패킷데이터의 '일부'일 수가 있다.  
![Buffer](/assets/img/posts/2026-02-27-Network-IOCP-GameServer-img1.svg)
이렇게 되면 배열 맨앞의 80bytes를 처리한 다음, 그 뒤의 20bytes는 배열 맨 앞으로 당겨 놓아야 한다. 그래야 수신버퍼의 공간이 모자라지 않은채로 나머지 데이터들을 받을 수 있기 때문이다.  
데이터를 배열 맨 앞으로 당겨놓는 작업이 그다지 좋아보이지는 않기 때문에 링버퍼를 사용한다.  
![Buffer](/assets/img/posts/2026-02-27-Network-IOCP-GameServer-img2.svg)


<details markdown="1">
<summary>링버퍼 코드 보기</summary>  

링버퍼 헤더
```cpp
class CRingbuffer
{
public:
  char* _buf;
  __int64 _size;

  char* _front;    // 데이터의 시작 위치
  char* _rear;     // 데이터의 마지막 위치 + 1

public:
  CRingbuffer(void);
  CRingbuffer(int size);
  ~CRingbuffer();

public:
  /* getter */
  int GetSize(void) { return (int)_size; }
  char* GetFront(void) { return _front; }
  char* GetRear(void) { return _rear; }
  char* GetBufPtr(void) { return _buf; }

  /* get size */
  int GetFreeSize(void);
  int GetUseSize(void);
  int GetDirectFreeSize(void);
  int GetDirectUseSize(void);

  /* enqueue, dequeue */
  int Enqueue(const char* src, int len);
  int Dequeue(char* dest, int len);
  int Peek(char* dest, int len);
  int Peek(char* dest, int len, int useSize, int directUseSize);

  /* pointer control */
  void MoveFront(int dist);
  void MoveRear(int dist);

  /* buffer control */
  void Clear(void);
  int Resize(int size);
};
```

링버퍼 cpp
```cpp
#include "CRingbuffer.h"
#include <iostream>

#define min(a, b)  (((a) < (b)) ? (a) : (b)) 
#define max(a, b)  (((a) > (b)) ? (a) : (b)) 

#define RINGBUFFER_DEFAULT_SIZE 10000
#define RINGBUFFER_MAX_SIZE     30000 

CRingbuffer::CRingbuffer(void)
{
  _size = RINGBUFFER_DEFAULT_SIZE;
  _buf = new char[_size + 1];
  _front = _buf;
  _rear = _buf;
}

CRingbuffer::CRingbuffer(int size)
{
  _size = size;
  _buf = new char[_size + 1];
  _front = _buf;
  _rear = _buf;
}

CRingbuffer::~CRingbuffer()
{
  delete[] _buf;
}




/* ringbuffer에 데이터를 쓴다.
현재 rear 위치부터 데이터를 쓰고, 쓰고 난 뒤 rear의 위치는 데이터의 마지막 위치 +1 이다.
@parameters
src : 데이터 포인터, len : 데이터 byte 길이
@return
성공 시 입력한 데이터 byte 크기를 리턴함.
ringbuffer에 충분한 공간이 없으면 실패하며, 0을 리턴함. */
int CRingbuffer::Enqueue(const char* src, int len)
{
  if (GetFreeSize() < len || len < 1)
    return 0;

  int directFreesize = GetDirectFreeSize();
  if (directFreesize < len)
  {
    memcpy(_rear, src, directFreesize);
    memcpy(_buf, src + directFreesize, len - directFreesize);
    _rear = _buf + len - directFreesize;
  }
  else
  {
    memcpy(_rear, src, len);
    _rear += len;
    if (_rear == _buf + _size + 1)
      _rear = _buf;
  }
  return len;
}




/* 현재 ringbuffer의 남은 공간 byte 크기를 얻는다.
@return
ringbuffer의 남은 공간 byte 크기 */
int CRingbuffer::GetFreeSize(void)
{
  volatile char* front = _front; // enqueue 스레드 1개, dequeue 스레드 1개인 멀티스레드 환경에서 FreeSize를 안전하게 계산하기 위해 front를 volatile로 선언함
  if (front > _rear)
    return (int)(front - _rear - 1);
  else
    return (int)(_size + front - _rear);
}


/* 현재 ringbuffer에서 사용중인 공간 byte 크기를 얻는다.
@return
ringbuffer의 사용중인 공간 byte 크기 */
int CRingbuffer::GetUseSize(void)
{
  volatile char* rear = _rear;
  if (_front > rear)
    return (int)(_size + rear - _front + 1);
  else
    return (int)(rear - _front);
}

/* 현재 ringbuffer에서 rear와 버퍼의 끝 사이의 남은 공간 byte 크기를 얻는다.
@return
ringbuffer에서 rear와 버퍼의 끝 사이의 남은 공간 byte 크기 */
int CRingbuffer::GetDirectFreeSize(void)
{
  volatile char* front = _front;
  if (front > _rear)
    return (int)(front - _rear - 1);
  else
  {
    if (front != _buf)
      return (int)(_buf + _size - _rear + 1);
    else
      return (int)(_buf + _size - _rear);
  }

}


/* 현재 ringbuffer에서 front와 버퍼의 끝 사이의 사용중인 공간의 byte 크기를 얻는다.
@return
ringbuffer에서 front와 버퍼의 끝 사이의 사용중인 공간 byte 크기 */
int CRingbuffer::GetDirectUseSize(void)
{
  volatile char* rear = _rear;
  if (_front > rear)
    return (int)(_buf + _size - _front + 1);
  else
    return (int)(rear - _front);
}





/* ringbuffer의 데이터를 dest로 읽는다.
현재 front 위치부터 데이터를 읽고, 읽은만큼 front를 전진시킨다.
@parameters
dest : 데이터 읽기 버퍼, len : 버퍼 byte 길이
@return
성공 시 읽은 데이터 byte 크기를 리턴함.
ringbuffer 내에 len 크기만큼 데이터가 있을 경우 len 만큼 읽음.
ringbuffer 내에 len 크기만큼 데이터가 없을 경우 남아있는 모든 데이터를 읽음. */
int CRingbuffer::Dequeue(char* dest, int len)
{
  if (len < 1)
    return 0;

  int useSize = GetUseSize();
  if (useSize == 0)
    return 0;

  int readBytes = min(len, useSize);
  int directUseSize = GetDirectUseSize();
  if (directUseSize < readBytes)
  {
    memcpy(dest, _front, directUseSize);
    memcpy(dest + directUseSize, _buf, readBytes - directUseSize);
    _front = _buf + readBytes - directUseSize;
  }
  else
  {
    memcpy(dest, _front, readBytes);
    _front += readBytes;
    if (_front == _buf + _size + 1)
      _front = _buf;
  }
  return readBytes;
}




/* ringbuffer의 데이터를 dest로 읽는다.
현재 front 위치부터 데이터를 읽고, front는 전진시키지 않는다.
@parameters
dest : 데이터 읽기 버퍼, len : 버퍼 byte 길이
@return
성공 시 읽은 데이터 byte 크기를 리턴함.
ringbuffer 내에 len 크기만큼 데이터가 있을 경우 len 만큼 읽음.
ringbuffer 내에 len 크기만큼 데이터가 없을 경우 남아있는 모든 데이터를 읽음. */
int CRingbuffer::Peek(char* dest, int len)
{
  if (len < 1)
    return 0;

  int useSize = GetUseSize();
  if (useSize == 0)
    return 0;

  int readBytes = min(len, useSize);
  int directUseSize = GetDirectUseSize();
  if (directUseSize < readBytes)
  {
    memcpy(dest, _front, directUseSize);
    memcpy(dest + directUseSize, _buf, readBytes - directUseSize);
  }
  else
  {
    memcpy(dest, _front, readBytes);
  }
  return readBytes;
}



/* ringbuffer의 데이터를 dest로 읽는다. 이 때 현재 사용중인 크기를 계산하지 않고 미리 전달받는다.
현재 front 위치부터 데이터를 읽고, front는 전진시키지 않는다.
@parameters
dest : 데이터 읽기 버퍼, len : 버퍼 byte 길이
@return
성공 시 읽은 데이터 byte 크기를 리턴함.
ringbuffer 내에 len 크기만큼 데이터가 있을 경우 len 만큼 읽음.
ringbuffer 내에 len 크기만큼 데이터가 없을 경우 남아있는 모든 데이터를 읽음. */
int CRingbuffer::Peek(char* dest, int len, int useSize, int directUseSize)
{
  if (len < 1)
    return 0;

  if (useSize == 0)
    return 0;

  int readBytes = min(len, useSize);
  if (directUseSize < readBytes)
  {
    memcpy(dest, _front, directUseSize);
    memcpy(dest + directUseSize, _buf, readBytes - directUseSize);
  }
  else
  {
    memcpy(dest, _front, readBytes);
  }
  return readBytes;
}



/* front의 위치를 len 만큼 뒤로 옮긴다.
@parameters
len : front의 위치를 뒤로 옮길 byte 크기 */
void CRingbuffer::MoveFront(int dist)
{
  _front = _buf + (_front - _buf + dist) % (_size + 1);
}

/* rear의 위치를 len 만큼 뒤로 옮긴다.
@parameters
len : rear의 위치를 뒤로 옮길 byte 크기 */
void CRingbuffer::MoveRear(int dist)
{
  _rear = _buf + (_rear - _buf + dist) % (_size + 1);
}


/* 버퍼를 비운다. */
void CRingbuffer::Clear(void)
{
  _front = _buf;
  _rear = _buf;
}

/* 버퍼 크기를 늘리거나 줄인다.
@parameters
size : 늘리거나 줄일 버퍼의 크기
@return
늘린 뒤의 버퍼 크기. 늘릴 수 있는 최대 크기는 RINGBUFFER_MAX_SIZE 이다.
*/
int CRingbuffer::Resize(int size)
{
  int newSize = min(max(1, size), RINGBUFFER_MAX_SIZE);
  char* newBuf = new char[newSize + 1];
  int readBytes = Dequeue(newBuf, newSize);

  delete _buf;
  _size = newSize;
  _buf = newBuf;
  _front = newBuf;
  _rear = newBuf + readBytes;

  return newSize;
}
```
</details>



## 2. 소켓에 Send할 때는 별도의 송신버퍼를 사용하지 않고 패킷자체를 송신버퍼로 사용한다.
어떤 패킷을 broadcast 해야한다고 하자.  
만약 소켓마다 별도의 송신버퍼를 사용한다면, broadcast 해야하는 패킷을 모든 소켓의 송신버퍼에 복사해 넣어야 한다.  
그런데 패킷 자체를 송신버퍼로 취급하여 모든 소켓이 같이 사용한다고 하면 패킷 복사 비용을 절약할 수 있다.  
![Buffer](/assets/img/posts/2026-02-27-Network-IOCP-GameServer-img3.svg)


## 3. 소켓에 Receive 요청은 반드시 1번씩만 한다.
WSARecv 함수로 소켓에 receive 요청을 할 때 일반적으로 비동기(Overlapped I/O)로 요청한다.  
그런데 비동기로 요청할 수 있다고 해서 receive 요청을 여러개를 동시에 해도 된다는 것은 아니다.  

소켓에 WSARecv 함수를 2번연속 호출하였다고 하자.  
그런데 소켓에 100bytes가 전달되어 첫번째 완료통지가 왔고, 곧바로 50bytes가 전달되어 두번째 완료통지가 왔다.  
IOCP worker 스레드는 멀티스레드로 완료통지를 처리하기 때문에 2개의 worker 스레드가 깨어나서 각각 100bytes, 50bytes 데이터를 전달받았다.  
그런데 TCP 특성상 쪼개진 패킷데이터를 받을수도 있는데, 이렇게 되면 패킷데이터가 모두 왔는지 아닌지 알수가 없다.  
그리고 처리도 매우 복잡해진다.  

소켓에 WSARecv가 1개 시점에 1번만 요청되도록 하려면 아래 순서대로 하면 된다.
1. 처음 Accept 했을 때 WSARecv를 호출한다.
2. receive 완료통지를 받았을 때 받은 패킷을 처리하고 다시 WSARecv를 호출한다.

## 4. 소켓에 Send 요청은 패킷을 모아서 한번에 보낸다.
보낼 패킷이 생길 때마다 WSASend를 호출하여 Send하면 문제가 있을까?  
기능상으로는 문제가 없다. 하지만 성능상으로는 문제가 있다.  
아래 간단한 시간측정 테스트 결과를 확인해보자.  

| 작업                      | 평균시간(마이크로초) |
| ------------------------- | -------------------- |
| WSASend 호출              | 103.0                |
| WSARecv 호출              | 15.1                 |
| 메모리 동적할당(1024byte) | 0.3                  |

실행되는 로직이 천차만별이라 직접적인 비교는 힘들지만, WSASend는 확실히 다른 작업에 비해 오래 걸리는 작업이다.  
WSASend를 호출하면 우선 커널모드에 진입해야하고, 보낼 데이터가 담인 유저영역의 메모리page(패킷 객체가 위치한 메모리page)를 lock 걸어야 하기 때문이다.  
그래서 서버의 성능을 높이기 위해서는 WSASend 함수의 호출 횟수를 줄이는 것이 관건이다.  

전략은 다음과 같다:
- 보낼 패킷을 모아두었다가 WSASend 함수를 한번 호출할 때 한번에 보낸다.

그런데 이렇게되면 또다른 문제가 발생한다. 패킷을 얼마나 모아서 보낼 것인가?  
이 문제까지 해결한 전략은 다음과 같다:
1. 패킷을 보내려 할 때
   1. 전송할 패킷을 소켓의 Send Queue에 삽입한다.
   2. 만약 소켓에 진행중인 Send 작업이 없다면 WSASend 함수를 호출하여 Send Queue에 있는 모든 패킷을 전송한다.
   3. 만약 소켓에서 Send 작업이 진행중이라면(= WSASend 함수에 대한 완료통지가 아직 안왔음) 아무것도 하지 않는다.

2. WSASend 호출에 대한 완료통지를 받았을 때
   1. 소켓의 Send Queue를 확인하여 보낼 데이터가 있다면 WSASend 함수를 호출하여 Send Queue에 있는 모든 패킷을 전송한다.
   2. 소켓의 Send Queue에 데이터가 없다면 아무것도 하지 않는다.

## 5. 세션(Session)의 생명주기를 잘 관리하라.
세션은 게임서버에서 소켓 사용에 필요한 여러 데이터들을 모아놓은 클래스이다.  
게임서버는 일반적으로 세션을 만들어 클라이언트를 관리한다.  
세션은 대략적으로 아래와 비슷하게 만들 수 있다:
```cpp
// 세션 클래스
class Session : public std::enable_shared_from_this<Session>
{
private:
  unsigned __int64 m_sessionId;  // 세션ID
  wchar_t m_szIP[16];            // IP
  unsigned short m_port;         // port
  SOCKET m_sock;   // 소켓

  ThreadSafeQueue<CPacket*> m_sendQueue;  // 전송할 패킷 모아놓은 큐  
  Ringbuffer m_recvQueue;   // 데이터 수신용 링버퍼

  OVERLAPPED_EX m_sendOverlapped;  // Send용 OVERLAPPED 구조체
  OVERLAPPED_EX m_recvOverlapped;  // Receive용 OVERLAPPED 구조체

public:
  std::shared_ptr<Session> GetPtr()
  {
    return shared_from_this();
  }
};

// OVERLAPPED 구조체를 확장한 OVERLAPPED_EX 구조체
struct OVERLAPPED_EX
{
  OVERLAPPED overlapped;   // 비동기IO를 위한 OVERLAPPED 구조체
  int ioType;              // 비동기IO가 Send인지 Receive인지 구분하는 값
  std::shared_ptr<Session> spSession;  // 비동기IO 도중에 세션객체가 제거되는것을 막기 위한 스마트포인터
}
```

그런데 소켓과 IOCP를 연결지을때 사용하는 CreateIoCompletionPort 함수를 살펴보자.
```cpp
// listen socket에서 accept 하여 클라이언트 소켓을 얻었다.
SOCKET socket = accept(listenSock, (SOCKADDR*)&addr, &lenAddr);

// 세션객체 생성
std::shared_ptr<Session> spSession = std::make_shared<Session>(socket);
Session* pSession = spSession.get();  // 세션객체 포인터 얻기

// 소켓과 IOCP를 연결지음
CreateIoCompletionPort(
  (HANDLE)socket, 
  hIOCP, 
  (ULONG_PTR)pSession,   // 세션의 포인터 전달
  0);
```
소켓과 IOCP를 연결지을 때는 CompletionKey로 8bytes의 데이터만 전달할 수 있기 때문에 세션의 포인터를 전달할 수밖에 없다.  
그런데 세션은 스마트포인터로 사용되고 있는데, 세션의 포인터를 전달해도 안전할까?  
잘못하면 소켓의 완료통지를 받았을 때 세션객체가 제거되어있을 수도 있다.  

소켓에 I/O 작업이 진행되는 도중에 세션객체가 제거되지 않기 위해 아래 전략을 사용한다:
1. 세션 매니저를 만들어 세션 스마트포인터를 유지시킨다.
```cpp
// 세션매니저 싱글톤 클래스
class SessionManager : public Singleton
{
  // 세션 스마트포인터를 가지고있는 map. 
  // 세션이 생성될 때 여기에 insert 하고, 세션의 소켓을 close할 때 여기에서 erase 한다.
  std::unordered_map<unsigned __int64, std::shared_ptr<Session>> m_sessionMap;
}
```
2. OVERLAPPED 구조체에 세션 스마트포인터를 입력하여 사용한다.  
세션 클래스를 보면, OVERLAPPED 구조체를 확장한 OVERLAPPED_EX 구조체를 사용하고 있다.  
여기에 세션의 스마트포인터를 입력할 수 있다.  
WSASend, WSARecv 함수로 비동기IO를 시작하기 전 세션 스마트포인터의 ref count를 1 높인다.  
그리고 비동기IO가 완료되면 ref count를 1 낮춘다.  
그래서 비동기IO 도중에 세션이 제거되지 않도록 한다.  

```cpp
// WSASend를 호출하기 전에 m_sendOverlapped->spSession에 세션 입력하여 ref count +1 하여 사용
spSession->m_sendOverlapped->spSession = spSession;
WSASend(spSession->sock, wsaBuf, 1, NULL, &flag, (OVERLAPPED*)&spSession->m_sendOverlapped, NULL);
```

```cpp
// Worker 스레드가 완료통지를 받음
DWORD numByteTrans;
ULONG_PTR completionKey;
OVERLAPPED_EX* pOverlapped;
CSession* pSession;
GetQueuedCompletionStatus(_hIOCP, &numByteTrans, &completionKey, (OVERLAPPED**)&pOverlapped, INFINITE);

// 세션 포인터를 얻는다.
pSession = (CSession*)completionKey;

if( pOverlapped->ioType == SEND )
{
  // Send 완료통지 처리로직...

  // Send 완료통지를 처리한 다음 OVERLAPPED 에서 세션을 제거하여 ref count -1
  pSession->m_sendOverlapped->spSession.reset();
}
```


