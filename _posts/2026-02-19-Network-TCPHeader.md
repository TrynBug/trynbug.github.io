---
title: TCP 헤더
date: 2026-02-19 15:42:40 +0900
categories: [네트워크, TCP]
tags: [network, TCP]
description: TCP 헤더
---

## TCP Segment

TCP 패킷은 네트워크 전송에서 데이터의 신뢰성을 보장하기 위해 복잡한 구조를 가지고 있다.  
TCP 패킷은 TCP 세그먼트(Segment) 라고 부른다.  
헤더(Header)와 데이터(Data/Payload) 부분으로 나뉜다.  
데이터는 사용자 어플리케이션에서 사용되는 데이터이다.

![TCP Header](/assets/img/posts/2026-02-19-Network-TCPHeader-img.png)
_(https://www.pynetlabs.com/transmission-control-protocol-tcp-header/)_

TCP 헤더의 최소 크기는 20byte 이다. (그림에서 Options를 제외한 모든 필드 크기를 더하면 20byte 이다)  
헤더에 Options가 추가될 경우 최대 60byte까지 늘어날 수 있다.  
각각의 필드들을 알아보자.

#### 1. Source Port Number (16bit)
패킷을 보내는 어플리케이션의 포트 번호이다.

#### 2. Destination Port Number (16bit)
목적지 어플리케이션의 포트 번호이다.

#### 3. Sequence Number (32bit)
처음 TCP 연결이 이루어지면, Sequence Number는 랜덤 4byte 값으로 시작한다.  
이 랜덤 값은 나와 상대방이 각자의 랜덤값을 생성하여 시작하며, 각자가 자신의 값을 관리한다.  
Sequence Number 값은 1byte를 전송할 때마다 값이 1씩 증가한다.  
Sequence Number 값은 항상 증가하기 때문에 패킷이 순서에 맞게 잘 전달되었는지, 중복된 패킷이 전달되지 않았는지 등을 체크하는데 사용된다.  

#### 4. Acknowledgment Number (32bit)
이 값은 상대방에게 내가 데이터를 어디까지 받았는지를 알려주는데 사용한다.  
만약 내가 상대방의 데이터를 Sequence Number = 100 까지 받았다면, 나는 상대방에게 Acknowledgment Number = 101 로 입력하여 패킷을 보낸다.  
그러면 상대방은 내가 Sequence Number = 100 까지는 데이터를 정상적으로 받았다는것을 알게되고, 다음 데이터를 적절히 전송해주게 된다.  
Acknowledgment Number 값은 상대방이 중복된 데이터를 재전송하지 않게 하여 네트워크 부하를 줄이는 역할을 한다.  

#### 5. DO (Data Offset) (4bit)
이 값은 TCP Segment에서 데이터(Data/Payload)가 시작되는 위치를 나타낸다.  
즉, TCP 헤더 크기를 나타낸다고 할 수 있다.  

#### 6. Reserved (3bit)
미래에 사용될 것으로 예상되어 비워둔 부분이다.

#### 7. Flags (9bit)
Flags 필드는 9bit 크기이며, 각각의 bit가 서로다른 의미를 가진다.  
주로 사용되는 bit들은 아래와 같다.  
- ACK (Acknowledgement)
  - ACK flag가 1로 세팅되면 TCP 헤더에서 Acknowledgment Number 값이 입력되었음을 나타낸다.
- SYN (Synchronize Sequence Number)
  - SYN flag가 1로 세팅되면 3-way handshake를 통해 TCP 연결을 하는 도중임을 나타낸다.
- RST (Reset)
  - RST flag는 TCP 연결을 즉시 끊을때 세팅한다.
  - RST flag가 1로 세팅되면 연결을 끊을때 4-way handshake를 거치지 않는다.
  - RST flag 없이 연결을 끊으려면 4-way handshake를 거쳐야 한다.
- FIN (Finish)
  - FIN flag는 4-way handshake로 연결을 종료할 때 사용하는 flag이다.
- PSH (Push Function)
  - PSH flag가 1로 세팅되면 현재 TCP 수신소켓의 데이터가 어플리케이션에 곧바로 전달되어야 함을 나타낸다.
  - TCP 패킷은 받자마자 즉시 어플리케이션에 전달되는것이 아닌데, 그 이유는 TCP 패킷의 데이터는 쪼개져서 전달될 수 있고, 쪼개진 데이터를 모두 받은게 아니라면 데이터를 어플리케이션에 전달하는 것은 의미가 없기 때문이다.
  - 아래와 같은 순서에서는 PSH 플래그가 유용하게 사용된다.  
    1. 수신측 어플리케이션이 TCP 소켓에 Recv 함수를 호출하였는데, 수신버퍼에 데이터가 없어서 스레드가 block 되었다.  
    2. 송신측에서 패킷을 전송하려 했는데 데이터가 커서 쪼개서 보내려 한다.
    3. 수신측 TCP가 패킷을 받았는데 쪼개진 데이터의 일부를 받았다(PSH 플래그 세팅안됨). 이 때에는 Recv를 기다리는 스레드가 깨어나지 않는다.
    4. 송신측에서 데이터를 쪼개서 보내다가 마지막 패킷에 PSH 플래그를 세팅하여 보냈다.
    5. 수신측 TCP가 패킷을 받았는데 PSH 플래그를 보고 마지막 데이터인것을 확인하여 Recv를 기다리는 스레드를 깨워서 데이터를 전달한다.
  - PSH 플래그를 사용함으로써 데이터가 모두 전달되지 않았는데도 스레드가 깨어나는 비효율적인 상황을 막을 수 있다.

#### 8. Window Size (16bit)
Window Size 값은 수신측의 수신버퍼 크기(byte단위)를 말한다.  
Window Size 값은 Window Scale 이라는 값과 조합되어 수신버퍼 크기를 나타낸다.  
만약 Window Size = 1000, Window Scale = 7 이라고 한다면 수신측의 수신버퍼 크기는 1000 * 2^7 = 128000byte 라는 의미이다.  
Window Scale 값은 TCP 헤더의 Options 필드로 전달되며, 최초 3-way handshake를 할 때만 1번 전달한다. 그래서 Window Scale 값은 중간에 바뀌지 않는다.  

Window Size 값은 TCP 통신 속도를 높이는데 특히 중요한 역할을 한다.  
만약 수신측이 자신의 수신버퍼 크기를 128000byte 라고 전달 했다면, 송신측은 보낼 데이터를 쉬지않고 연속으로 최대 128000byte 까지 보낼 수 있다.  

만약 128000byte 까지 보냈는데 수신측으로부터 아무런 응답이 없다면 송신측은 더이상 데이터를 보내지 않을 것이다.  
만약 128000byte 까지 보내는 도중인데, 수신측이 Acknowledgment Number 값을 통해 데이터를 어디까지 받았는지 알려준다면 송신측은 수신측의 남은 수신버퍼 크기를 계산하여 데이터를 추가로 더 보낼 수 있다.  

#### 9. Checksum (16bit)
Checksum 값은 데이터가 올바르게 전달되었는지 체크하는데 사용된다.  
이 값은 TCP segment의 모든 값을 더하여 계산된다.  
Checksum 값은 패킷을 보낼 때 NIC(네트워크 카드)가 계산해서 값을 채워넣고, 패킷을 받을 때 NIC가 Checksum 값을 다시 계산하여 검증한다.  
만약 중간에 네트워크 장비의 문제로 패킷의 bit가 변경되었다면 Checksum 검증 단계에서 걸러질 것이다.  

#### 10. Urgent Pointer (16bit)
Urgent Pointer 값은 패킷이 TCP 수신버퍼에서 대기하지 않고 받는 즉시 처리되어야 할 때 사용되는 값이다.  
일반적으로는 사용할 일이 없다.  


## Sliding Window 방식의 패킷전송
Sliding Window(슬라이딩 윈도우) 방식의 패킷전송은 TCP 프로토콜이 전송 속도를 조절하는데 사용하는 핵심 메커니즘이다.  
