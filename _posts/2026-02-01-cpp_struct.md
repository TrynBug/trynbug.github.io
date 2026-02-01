---
title: C++ 구조체
date: 2026-02-01 21:26:31 +0900
categories: [C++, Grammar]
tags: [c++, grammar]
description: C++ 구조체
---

## 요약
- 구조체의 멤버들 사이에는 padding이 있다.
- 구조체의 전체 크기는 크기가 가장 큰 멤버 크기의 배수이다. (예: 멤버 중 long long 멤버가 있다면 구조체의 크기는 8의 배수임)


## 1. 구조체의 멤버들 사이에는 padding이 있다.

아래와 같은 구조체가 있다.
```cpp
struct stData
{
    int a;     // 4 byte
    char b;    // 1 byte
    short c;   // 2 byte
    char d;    // 1 byte
    int e;     // 4 byte
    char f;    // 1 byte
    short g;   // 2 byte
};
```
이 구조체의 크기는 얼마일까?  
변수 타입 크기의 총 합이라면 4 + 1 + 2 + 1 + 4 + 1 + 2 = 15 byte 이어야 한다.  
그런데 실제 크기는 20 byte 이다.  
그 이유는 구조체의 멤버들 사이에는 아래와 같이 padding이 들어가있기 때문이다.
```cpp
struct stData
{
    int a;     // 4 byte
    char b;    // 1 byte
    (1 byte padding)
    short c;   // 2 byte
    char d;    // 1 byte
    (3 byte padding)
    int e;     // 4 byte
    char f;    // 1 byte
    (1 byte padding)
    short g;   // 2 byte
};
```

그런데 이는 구조체 내의 멤버 뿐만 아니라, 클래스 멤버, 지역변수 등의 모든 변수에도 해당되는 이야기이다.  
모든 변수는 메모리상에서 자기 자신 크기의 경계에 위치해야 한다.  
예를 들면, short 는 2바이트이기 때문에 메모리상에서 0바이트 위치, 2바이트 위치, 4바이트 위치, 6바이트 위치... 등에 위치할 수 있다.  
int 는 4바이트이기 때문에 메모리상에서 0바이트 위치, 4바이트 위치, 8바이트 위치... 등에 위치할 수 있다.  
int64 는 8바이트이기 때문에 메모리상에서 0바이트 위치, 8바이트 위치, 16바이트 위치... 등에 위치할 수 있다.  
  
그리고 구조체 전체의 크기는 구조체 멤버 중 크기가 가장 큰 멤버의 크기에 배수이어야 한다.  
예를 들면, 구조체 내에 int64 멤버가 하나라도 있다면, 해당 구조체의 크기는 8의 배수이어야 한다. 모자라는 크기는 구조체 맨 뒤에 padding 을 붙여 해결한다.  
stack 메모리 공간도 이것과 같은 규칙을 지켜야 한다.  
stack 메모리 공간의 크기는 stack 메모리 공간 내의 가장 큰 변수 크기의 배수이어야 한다.  
  
이렇게 되어야 하는 이유는 변수가 메모리상에서 자신 크기의 경계에 위치해있어야 연산이 원자적으로 수행되기 때문이다.


## 2. 변수가 메모리상에서 자신 크기의 경계에 위치해있지 않으면 발생하는 문제
2개 스레드가 아래 코드를 실행하고 있다.
```cpp
int a = 0;

// 1번 스레드
while(true)
{
    a = 0;
    a = 0xffffffff;
}

// 2번 스레드
while(true)
{
    b = a;  // 이 코드는 어셈블리 코드로 변환 시 mov 딱 하나가 될 것이다.
}
```

2번 스레드가 b에 a를 입력하고 있는데, 그러면 b의 값은 0 또는 0xffffffff 중 하나일까?  
a가 메모리상에서 4바이트 경계에 위치한다면 그렇다.  

하지만 만약 a가 메모리상에서 4바이트 경계에 위치하지 않는다면 b의 값은 0, 0x00ffffff, 0x0000ffff, 0x000000ff, 0xffffffff 등이 될 수 있다.  
그런데 a가 메모리상에서 4바이트 경계에 위치하지 않는다고 반드시 그런것은 아니고, a가 메모리상에서 2개의 cache line에 걸쳐 있어야 이렇게 된다.  
a가 메모리상에서 2개의 cache line에 걸쳐 있으면 CPU는 a 변수의 값을 write(또는 read)하기 위해 2개의 cache line을 수정해야 하고,  
그렇게 되면 1개의 mov 연산 조차 원자적이지 않게 된다.  
1개의 cache line은 일반적으로 64byte 이기 때문에 a가 메모리상에서 4byte 경계에 위치해 있다면 반드시 1개 cache line내에 위치하기 때문에 연산이 원자적으로 수행될수있다.  

그럼 실제로 이런 문제가 발생하는지 코드로 확인해보자.
```cpp
// 먼저 Data 구조체를 만든다. pragma pack(1) 지시어를 사용하여 구조체의 멤버들 사이에 padding이 없도록 한다.
#pragma pack(1)
struct Data
{
    char arr[62];   // 62byte 크기의 배열
    int a;          // 이 int 멤버는 메모리상에서 arr 바로 다음에 위치한다.
};
#pragma pack(pop)

int main()
{
    // Data 구조체를 메모리상에서 64byte 경계에 선언한다.
    // 이제 구조체의 arr 멤버는 메모리상에서 64byte 배수에 위치하지만, 
    // a 멤버는 arr멤버 바로 뒤에 있기 때문에 구조체 시작위치로부터 63~66byte 위치에 있게 된다. 
    // 즉, 2개의 cache line에 걸쳐서 있게 된다.
    Data alignas(64) data = { 0, };

    // 스레드1은 a에 0 또는 0xFFFFFFFF를 무한히 입력한다.
    std::thread thread1 = std::thread([&data]()
    {
        while (true)
        {
            data.a = 0;
            data.a = 0xFFFFFFFF;
        }
    });

    // 스레드2는 a 값을 읽어서 출력한다.
    std::thread thread2 = std::thread([&data]()
    {
        while (true)
        {
            int b = data.a;
            printf("%8X\n", b);
        }
    });
}
```
#### 프로그램 실행결과:
b의 값이 00000000, FFFF0000, 0000FFFF, FFFFFFFF 4개 중 하나로 나오고 있다.  
data.a에 값을 write 하는 작업이 원자적으로 수행되고 있지 않는것을 알 수 있다.  
![실행결과](/assets/img/posts/2026-02-01-cpp_struct_img1.png){: .left}  

<div style="clear: both;"></div>
