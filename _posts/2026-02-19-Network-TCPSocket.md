---
title: TCP 소켓
date: 2026-02-19 20:20:33 +0900
categories: [네트워크, TCP]
tags: [network, TCP]
description: TCP 소켓
---

## 3-way handshake
TCP 소켓은 3-way handshake 방식으로 연결을 수립한다.

![3-way handshake](/assets/img/posts/2026-02-19-Network-TCPSocket-img1.png)


## 4-way handshake
TCP 소켓은 3-way handshake 방식으로 연결을 끊는다.

![4-way handshake](/assets/img/posts/2026-02-19-Network-TCPSocket-img2.png)

## PC의 모든 소켓 상태 보기
netstat -an 명령어로 PC의 모든 소켓 상태를 볼 수 있다.

![netstat](/assets/img/posts/2026-02-19-Network-TCPSocket-img3.png)

LISTENING 상태의 소켓은 연결을 기다리고 있는 소켓이다.  
서버에서는 알 수 없는 프로세스가 LISTENING 소켓을 열어두고 있는것은 잠재적인 위협이 될 수 있다.  
TIME_WAIT 상태의 소켓이 아주 많다면 포트가 부족하여 클라이언트가 연결하지 못할수도 있다.  

## Sliding Window 방식의 패킷전송
Sliding Window(슬라이딩 윈도우) 방식의 패킷전송은 TCP 프로토콜이 전송 속도를 조절하는데 사용하는 핵심 메커니즘이다.  
TCP는 상대방의 Window Size(수신버퍼 크기)가 남은 만큼 계속해서 패킷을 전송한다.  
그래서 상대방의 ACK 응답을 오랫동안 기다리지 않고도 계속해서 패킷을 전송할 수 있다.  

![sliding window](/assets/img/posts/2026-02-19-Network-TCPSocket-img4.png)

Window Size 와 Window Scale 값 설명은 아래 포스트 참조:  
[TCP헤더]({% post_url 2026-02-19-Network-TCPHeader %})


## TCP 소켓 옵션 : SO_KEEPALIVE
SO_KEEPALIVE 소켓 옵션은 주기적으로 연결 상태를 확인하는 옵션이다.(대략 2시간 간격)  
TCP에서는 연결을 끊기 전 까지는 연결이 유지된 상태이고, 연결에 대한 정보가 메모리에 유지된다.  
상대방측의 랜선이 끊긴다고 해도 일단 연결 상태는 유지된다.  
상대방측에게 4~5회 통신을 시도해보고 응답이 없을 때가 되어서야 상대방과 연결이 끊겼다는 것을 알 수 있다. 이 때 연결을 끊고 RST 패킷를 보낸다.  
SO_KEEPALIVE 옵션은 이러한 연결 상태 확인 통신을 주기적으로 할지 여부를 결정한다.  

그런데 클라이언트 측 프로그램이 무한루프에 빠져 먹통이 되었다고 하자.  
클라이언트측의 OS가 중지된것이 아니라면 클라이언트측 TCP는 응답을 할 것이다. 연결상태 체크는 사용자 어플리케이션이 수행하는것이 아니기 때문이다  
그래서 SO_KEEPALIVE 옵션으로는 클라이언트가 멈춘 상태를 알아차리지 못한다.  

게임 서버에서 원하는 것은, L4(TCP)가 정상인지를 알아보는게 아니라 L7(클라이언트 어플리케이션)이 정상인지를 알아야 한다.  
그래서 클라이언트는 heartbeat 패킷을 서버에게 주기적으로 보내서 클라이언트가 살아있음을 알린다.  
일반적으로 대략 1분 주기로 heartbeat 패킷을 보내도록 구현한다.  

## TCP 소켓 옵션 : REUSEADDR
REUSEADDR 옵션은 IP 주소, 포트 번호 재사용 허용 여부를 설정하는데 사용한다.  
만약 어떤 포트에 할당된 소켓이 알 수 없는 이유로 LISTENING 상태이거나, FIN_WAIT_2 상태 등일 때 이 포트에 다시 bind하려 하면 오류가 발생한다.  
이 때 REUSEADDR 옵션을 지정하면 다시 bind해도 오류가 발생하지 않는다.  
이것은 현재 윈도우에서 랜덤 포트 할당 시에는 적용되지 않는다고 한다.  
포트를 직접 bind 할 때는 적용되는듯 하다.  
잘 사용하지는 않는다.  

## TCP 소켓 옵션 : SO_LINGER
SO_LINGER 옵션은 소켓의 연결을 끊을 때 4-way handshake를 하지 않고 RST 패킷으로 바로 끊고싶을때 사용한다.  

서버에서 4-way handshake로 연결을 끊으면 먼저 FIN 패킷을 보낸다.  
그런데 FIN 패킷을 보내 순서는 소켓 송신버퍼에 있는 데이터를 모두 보낸 다음이다.  
그래서 소켓을 닫는다고 해서 소켓 송신버퍼에 남아있는 데이터가 사라지는 것은 아니다(그런데 보내기는 하는데 상대방이 100% 다 받았는지는 알 수 없다).  

4-way handshake의 문제는 서버에서 FIN 패킷을 보낸 다음, 클라이언트가 FIN 패킷을 보내지 않으면 서버측의 소켓이 FIN_WAIT_2 상태로 계속 남아있다는 것이다.  
클라이언트측에 문제가 있다고 판단되어 서버가 연결을 끊는 상황이라면 클라이언트는 FIN 패킷을 보내지 못할 가능성이 높다.  
그러면 서버측의 소켓이 일정시간(대략2분)동안 FIN_WAIT_2 으로 남게 되고 잘못하면 포트번호가 부족해질 수 있다.  
이런 문제를 방지하고 싶다면 아래와 같이 SO_LINGER 옵션을 설정하면 연결을 끊을 때 4-way handshake를 하지 않고 바로 RST 패킷을 보내 연결을 끊는다.  
```cpp
LINGER linger;
linger.l_onoff = 1;
linger.l_linger = 0;
setsockopt(socket, SOL_SOCKET, SO_LINGER, (char*)&linger, sizeof(linger));
```

## TCP 소켓 옵션 : TCP_NODELAY
TCP_NODELAY 옵션은 Nagle 알고리즘 작동 여부를 설정한다.  
Nagle 알고리즘을 끄면 send하는 즉시 TCP에서 데이터를 보낸다.  
Nagle 알고리즘이 켜져 있으면 send한 데이터를 TCP 송신버퍼에 모아서 보낸다.  

sliding window 방식으로 패킷을 주고받는 상황을 생각해보자.  
송신측에서는 상대방의 window size를 체크하여 수신버퍼 크기가 0이 될 때 까지 데이터를 계속 보낼 것이다.  
수신측에서는 데이터를 일정횟수 받은 후에 ACK를 보내서 어디까지 데이터를 받았는지 알린다.  
송신측에서는 ACK가 언제 오는지를 계속 체크하는데, ACK가 늦게 오면 네트워크가 혼잡한 것으로 보고 전송을 느리게 한다.  
ACK가 빨리 오면 네트워크가 혼잡하지 않은 것으로 보고 전송을 빠르게 한다.  
ACK를 받았을 때, 송신버퍼의 데이터를 체크하여 Acknowledge Number 값보다 작은 Sequence Number를 가진 데이터(전송이 완료된 데이터)는 송신버퍼에서 삭제한다.  
이것이 Nagle 알고리즘이 꺼졌을 때 작동방식이다.  

Nagle 알고리즘이 켜져있을 때는 작동 방식이 약간 다르다.  
멘 처음에 송신버퍼가 비어있을 때는 send할 데이터를 바로 전송한다.  
맨 처음 전송한다음 그다음 데이터들은 송신버퍼에 모아둔다.  
모으는 도중에 상대방에게서 ACK를 받으면, 송신버퍼에 모아놓은 데이터를 전송한다.  
또는 데이터를 모으다가 MSS 크기에 도달하면 그 때도 모은 데이터를 전송한다.  
만약 데이터를 보내서 송신버퍼가 비어있을 때 상대방에게서 ACK를 받았다면 다음 send하는 데이터는 즉시 전송된다.  
이렇게 데이터를 전송할 때도 당연히 상대방의 window size를 체크하여 수신버퍼 크기가 0이라면 데이터를 보내지 않는다.  

Nagle 알고리즘을 껐을 때와 켰을 때를 비교해보면, Nagle을 껐을 때 패킷이 실제로 전송되는 횟수가 더 많다.  
패킷을 전송하려면 데이터 크기와는 상관없이 TCP 헤더(20~60byte)가 필요하기 때문에, 패킷 전송 횟수가 많아지면 트래픽이 증가한다.  
그래서 Nagle을 킨다면 트래픽이 줄어들어 네트워크 혼잡도를 줄일 수 있다.  

게임 서버 입장에서 Nagle 알고리즘 사용 여부를 고려하면 다음과 같다:
- FPS 게임, 키보드 방향키로 캐릭터의 움직임을 빠르게 조작해야하는 게임 : Nagle 알고리즘을 꺼서 전송속도를 높인다.
- 일반적인 RPG 게임 : Nagle 알고리즘을 켜서 트래픽을 줄인다.

## graceful shutdown
closesocket은 일반적인 연결끊기이다. FIN을 보내고, 소켓을 반환한다.  
graceful shutdown은 연결을 끊을 때 데이터 유실없이 안전하게 끊고 싶을때 사용한다.  
그런데 사실 게임서버 입장에서는 graceful shutdown을 사용할 필요가 없다.  
왜냐하면 연결을 끊을 때 마지막 데이터가 전달 안된다고 해서 별일이 일어나지는 않기 때문이다.  

```cpp
// SD_SEND 옵션으로 shutdown 하면 상대방에게 '나는 더이상 데이터를 보내지 않음' 이라는 의미이다.
// shutdown 하면 소켓 송신버퍼의 데이터를 모두 보내고 상대방에게 FIN을 보낸다. 
// shutdown 후에 소켓에 send 함수를 호출하면 오류가 발생한다.
// 상대방에게서 FIN을 받은 뒤에 소켓을 반환한다.
shutdown(socket, SD_SEND);
```

## 소켓 옵션의 상속
소켓에 적용해둔 옵션은 이 소켓을 통해 생성된 다른 소켓에도 상속된다.
그래서 Accept 소켓(Listen 소켓)에 옵션을 1번만 지정해 놓으면 된다.

## backlog queue
int listen(SOCK s, int backlog);
backlog 는 서버가 당장 처리하지 않더라도 연결 가능한 클라이언트의 개수이다.  
TCP 연결할 때의 3-way handshake는 사용자 어플리케이션이 수행하는게 아니라 TCP가 수행하는 것이다.  
그래서 TCP 연결이 성공했을 때 연결정보를 OS가 connection queue(backlog queue)에 저장해두었다가 사용자 어플리케이션이 accept 함수를 호출했을 때 연결정보를 하나씩 전달해준다.  

backlog queue의 크기를 최대로 설정하려면 SOMAXCONN(0x7fffffff) 을 입력하면 되는데, 그러면 크기가 200으로 세팅된다.  
서버측의 backlog queue가 가득 찼을 때 클라이언트가 연결하려고 하면 클라이언트측에서 WSAECONNREFUSED 10061 오류가 발생한다.  
만약 클라이언트 PC에서 IP에 할당할 사설 포트 번호가 부족하면 WSAENOBUFS 10055 오류가 발생한다.  

backlog queue 크기는 굳이 크게 잡을 필요 없는데, 서버에서 계속해서 accept 한다면 1개 스레드로도 초당 수천건의 연결을 처리할 수 있기 때문이다.

```cpp
// listen 소켓 생성
SOCKET listenSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

// listen() 함수를 호출할 때 backlog queue 크기를 지정할 수 있다.
int backlogSize = 100;
listen(listenSocket, SOMAXCONN_HINT(backlogSize));
```
