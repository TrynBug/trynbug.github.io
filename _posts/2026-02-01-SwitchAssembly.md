---
title: C++ switch문 Assembly 분석
date: 2026-02-01 01:00:00 +0900
categories: [C++, Assembly]
tags: [c++, assembly]
description: C++ switch문 Assembly 분석
---

## 요약
- switch 문은 case를 순서대로 작성할 경우 컴파일러가 jump 테이블을 만들어 최적화해준다.
- case를 순서대로 작성하지 않는다면 컴파일러가 최적화해주지 않는다.
- switch 문은 반드시 case를 순서대로 작성하도록 하자.


## 1. switch 문과 jump 테이블
아래 switch문의 assembly를 분석해보자.
```cpp
int MessageType = 0;
int Data = 0;
switch (MessageType)
{
case 0:
    Data = 10;
    break;
case 1:
    Data = 20;
    break;
case 2:
    Data = 30;
    break;
case 3:
    Data = 40;
    break;
}
```
여기서 주목할 점은 switch 문의 case가 0, 1, 2, 3 순서대로 작성되어 있는 것이다.  
switch 문은 기본적으로 if문과 다르지 않으나, case 값이 순서대로 작성되어 있다면 컴파일러는 jump 테이블을 생성하여 코드를 좀 더 효율적으로 작성해준다.  
어떻게 효율적으로 작성하는지 확인해보자.

assembly
```cpp
    switch (MessageType)
0x00CB10D0  mov         eax,dword ptr [MessageType]        // MessageType 주소에 있는 값을 eax에 넣음
0x00CB10D3  mov         dword ptr [ebp-28h],eax            // eax에 있는 값(MessageType)을 ebp-28h 주소에 넣음(현재 esp는 ebp-2Ch 임)
0x00CB10D6  cmp         dword ptr [ebp-28h],3              // ebp-28h 주소에 있는 값(eax, MessageType)을 3과 비교함
0x00CB10DA  ja          main+88h (0CB1108h)                // 값이 크다면(ja, Jump Above) 0CB1108h 주소로 jump함. 이 주소는 switch 문이 끝난 다음 주소임.
0x00CB10DC  mov         ecx,dword ptr [ebp-28h]            // ebp-28h 주소에 있는 값(eax, MessageType)을 ecx 에 넣음
0x00CB10DF  jmp         dword ptr [ecx*4+0CB1250h]         // ecx*4+0CB1250h 주소에 있는 값으로 jump함. 이 주소 위치에는 각 case 문 시작 주소에 해당하는 주소값이 들어있음
    {
    case 0:
        Data = 10;
0x00CB10E6  mov         dword ptr [Data],0Ah
        break;
0x00CB10ED  jmp         main+88h (0CB1108h)
    case 1:
        Data = 20;
0x00CB10EF  mov         dword ptr [Data],14h
        break;
0x00CB10F6  jmp         main+88h (0CB1108h)
    case 2:
        Data = 30;
0x00CB10F8  mov         dword ptr [Data],1Eh
        break;
0x00CB10FF  jmp         main+88h (0CB1108h)
    case 3:
        Data = 40;
0x00CB1101  mov         dword ptr [Data],28h
        break;
    }
```
assembly를 분석해보면, case에 해당하는 값이 아닐 경우에는 0CB1108h 주소로 jump 한다. 이 위치는 switch문이 끝난 다음의 코드 주소이다.  
그런데 case에 해당하는 값일 때에는 ecx*4+0CB1250h 주소로 jump 한다. 이 위치에 있는 메모리값을 확인해보자
```cpp
// 0x00CB1250 주소에 있는 메모리 값(jump 테이블) :
0x00CB1250  e6 10 cb 00      // 0x00CB10E6; case 0 에 해당하는 명령어 위치
0x00CB1254  ef 10 cb 00      // 0x00CB10EF; case 1 에 해당하는 명령어 위치
0x00CB1258  f8 10 cb 00      // 0x00CB10F8; case 2 에 해당하는 명령어 위치  
0x00CB125C  01 11 cb 00      // 0x00CB1101; case 3 에 해당하는 명령어 위치
```
ecx 레지스터에는 MessageType 변수값이 들어있었고, 0CB1250h 주소의 메모리에는 jump 테이블(컴파일러가 자동으로 생성함)이 입력되어 있었다.  
그리고 MessageType * 4 + 0CB1250h 를 계산하여 해당 주소의 메모리를 확인해보면 case 0, 1, 2, 3 에 해당하는 명령어 주소가 입력되어 있는것을 확인할 수 있다.  
이 말은 switch문의 case가 몇 개가 있던 간에 단 한번의 연산으로 다음번 실행해야할 명령어 주소를 알 수 있다는 의미이다.  
이러한 컴파일러 최적화는 switch문의 속도를 크게 향상시킨다.



## 2. 컴파일러가 jump 테이블을 만들지 못하는 switch문

아래와 같이 case가 순서대로 작성되지 않거나 값의 범위가 매우 넓다면 컴파일러는 jump 테이블을 만들지 못한다.  
그러면 switch문은 if문과 같이 동작하게 된다.
```cpp
switch (MessageType)
{
case 1:
    Data = 10;
    break;
case 99999:
    Data = 20;
    break;
case 30:
    Data = 30;
    break;
case 180:
    Data = 40;
    break;
}
```

assembly
```cpp
    switch (MessageType)
0x001C1149  mov         edx,dword ptr [MessageType]      // MessageType 주소에 있는 값을 edx에 넣음
0x001C114C  mov         dword ptr [ebp-8],edx            // edx 값을 ebp-8 주소에 넣음
0x001C114F  cmp         dword ptr [ebp-8],0B4h           // ebp-8 주소에 있는 값을 B4h(180) 과 비교함
0x001C1156  jg          main+0EFh (01C116Fh)             // Jump if (signed) Greater. ebp-8 주소에 있는 값(MessageType)을 부호있는 값으로 보았을 때, B4h(180) 보다 크다면 001C116Fh 주소로 jump 한다. 이 주소는 case 99999 를 검사하기 위한 주소임.
0x001C1158  cmp         dword ptr [ebp-8],0B4h           // ebp-8 주소에 있는 값을 B4h(180) 과 비교함
0x001C115F  je          main+115h (01C1195h)             // Jump if Equal. 값이 동일할 경우 01C1195h 주소로 jump함. 이 주소는 case 180에 해당하는 주소임.
0x001C1161  cmp         dword ptr [ebp-8],1              // ebp-8 주소에 있는 값을 1 과 비교함
0x001C1165  je          main+0FAh (01C117Ah)             // 값이 동일할 경우 01C117Ah 주소로 jump함. 이 주소는 case 1에 해당하는 주소임.
0x001C1167  cmp         dword ptr [ebp-8],1Eh            // ebp-8 주소에 있는 값을 1Eh(30) 과 비교함
0x001C116B  je          main+10Ch (01C118Ch)             // 값이 동일할 경우 01C118Ch 주소로 jump함. 이 주소는 case 30에 해당하는 주소임.
0x001C116D  jmp         main+11Ch (01C119Ch)             // (위 비교식이 다 실패한 경우) 01C119Ch 주소로 jump함. 이 주소는 swtich 문 밖 주소임.
0x001C116F  cmp         dword ptr [ebp-8],1869Fh         // ebp-8 주소에 있는 값을 1869Fh(99999) 와 비교함
0x001C1176  je          main+103h (01C1183h)             // 값이 동일할 경우 01C1183h 주소로 jump함. 이 주소는 case 99999에 해당하는 주소임.
0x001C1178  jmp         main+11Ch (01C119Ch)             // (위 비교식이 다 실패한 경우) 01C119Ch 주소로 jump함. 이 주소는 swtich 문 밖 주소임.
    {
    case 1:
        Data = 10;
0x001C117A  mov         dword ptr [Data],0Ah
        break;
0x001C1181  jmp         main+11Ch (01C119Ch)
    case 99999:
        Data = 20;
0x001C1183  mov         dword ptr [Data],14h
        break;
0x001C118A  jmp         main+11Ch (01C119Ch)
    case 30:
        Data = 30;
0x001C118C  mov         dword ptr [Data],1Eh
        break;
0x001C1193  jmp         main+11Ch (01C119Ch)
    case 180:
        Data = 40;
0x001C1195  mov         dword ptr [Data],28h
        break;
    }
```



## 3. case의 범위가 넓은 switch문
case의 범위가 적당히 넓을 경우에는 컴파일러가 jump 테이블을 만들어준다.  

아래와 같은 switch 문이 있다.
```cpp
switch (MessageType)
{
case 1:
    Data = 10;
    break;
case 5:
    Data = 20;
    break;
case 30:
    Data = 30;
    break;
case 46:
    Data = 40;
    break;
}
```

assembly
```cpp
// 컴파일러는 MessageType 값이 46 보다 크다면 밖으로 switch문 밖으로 나가도록 코드를 작성해준다.
// 그리고 MessageType 값을 1->0, 5->1, 30->2, 46->3 으로 변환하고, MessageType 값이 2~4, 6~29, 31~45 인 경우 4로 만드는 테이블을 하나 더 만든다.
// 그리고 case 값 0~3 을 기준으로 jump 테이블을 만든다.

    switch (MessageType)
0x001C119C  mov         eax,dword ptr [MessageType]         // MessageType 주소에 있는 값을 eax에 넣음
0x001C119F  mov         dword ptr [ebp-14h],eax             // eax를 ebp-14h 주소에 넣음
0x001C11A2  mov         ecx,dword ptr [ebp-14h]             // ebp-14h 주소에 있는 값을 ecx에 넣음
0x001C11A5  sub         ecx,1                               // ecx(MessageType)에서 1을 뺌
0x001C11A8  mov         dword ptr [ebp-14h],ecx             // ecx(MessageType-1) 값을 ebp-14h 주소에 넣음
0x001C11AB  cmp         dword ptr [ebp-14h],2Dh             // ebp-14h 주소의 값을 2Dh(45) 와 비교함
0x001C11AF  ja          $LN43+7h (01C11E4h)                 // ebp-14h 주소에 있는 값이 더 크다면,  01C11E4h 로 jump함. 이 주소는 switch문 밖임
0x001C11B1  mov         edx,dword ptr [ebp-14h]             // ebp-14h 주소에 있는 값(MessageType-1)을 edx에 넣음
0x001C11B4  movzx       eax,byte ptr [edx+1C1284h]          // edx+1C1284h 주소에 있는 값을 eax 에 넣음. 1C1284h 주소에는 값 치환 테이블이 있음. movzx 는 부호가 없는 8 또는 16비트 source 값을 부호가 없는 32비트로 확장하여 레지스터에 넣을 때 사용한다. 
0x001C11BB  jmp         dword ptr [eax*4+1C1270h]           // eax*4+1C1270h 주소에 있는 값으로 jump 함. 1C1270h 주소에는 jump 테이블이 있음.
    {
    case 1:
        Data = 10;
0x001C11C2  mov         dword ptr [Data],0Ah
        break;
0x001C11C9  jmp         $LN43+7h (01C11E4h)
    case 5:
        Data = 20;
0x001C11CB  mov         dword ptr [Data],14h
        break;
0x001C11D2  jmp         $LN43+7h (01C11E4h)
    case 30:
        Data = 30;
0x001C11D4  mov         dword ptr [Data],1Eh
        break;
0x001C11DB  jmp         $LN43+7h (01C11E4h)
    case 46:
        Data = 40;
0x001C11DD  mov         dword ptr [Data],28h
        break;
    }

// 0x001C1284 메모리 주소 값(값 치환 테이블)
0x001C1284  00 04 04 04     //  0→0  1→4  2→4  3→4
0x001C1288  01 04 04 04     //  4→1  5→4  6→4  7→4
0x001C128C  04 04 04 04     //  8→4  9→4 10→4 11→4
0x001C1290  04 04 04 04     // 12→4 13→4 14→4 15→4
0x001C1294  04 04 04 04     // 16→4 17→4 18→4 19→4
0x001C1298  04 04 04 04     // 20→4 21→4 22→4 23→4
0x001C129C  04 04 04 04     // 24→4 25→4 26→4 27→4
0x001C12A0  04 02 04 04     // 28→4 29→2 30→4 31→4
0x001C12A4  04 04 04 04     // 32→4 33→4 34→4 35→4
0x001C12A8  04 04 04 04     // 36→4 37→4 38→4 39→4
0x001C12AC  04 04 04 04     // 40→4 41→4 42→4 43→4
0x001C12B0  04 03 3b 0d     // 44→4 45→3

// 0x001C1270 메모리 주소 값(jump 테이블)
0x001C1270  c2 11 1c 00     // case 1 에 해당하는 주소
0x001C1274  cb 11 1c 00     // case 5 에 해당하는 주소
0x001C1278  d4 11 1c 00     // case 30 에 해당하는 주소
0x001C127C  dd 11 1c 00     // case 46 에 해당하는 주소
```

