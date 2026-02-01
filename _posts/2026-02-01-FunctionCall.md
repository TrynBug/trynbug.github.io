---
title: C++ 함수호출 Assembly 분석
date: 2026-02-01 14:57:42 +0900
categories: [C++, Assembly]
tags: [c++, function, assembly]
description: C++ 함수호출 Assembly 분석
---

## 요약
- 함수가 호출될 때 함수 파라미터는 stack 영역에 저장되어 함수로 전달된다. 일반적으로 가장 오른쪽 파라미터부터 가장 왼쪽 파라미터 순서로 저장된다.
- `cdecl` 함수호출규약은 함수를 호출한 코드가 호출한 함수의 stack 메모리 해제 작업을 한다.
- `stdcall` 함수호출규약은 함수를 내부에서 stack 메모리 해제 작업을 한다.
- x64 플랫폼에서는 Windows의 경우에는 `Microsoft x64 Calling Convention`을 사용하고, Linux에서는 `System V AMD64 ABI` 를 사용하는데, 둘다 stack 메모리를 함수 호출자가 정리한다.


## 1. 함수 호출 분석

아래 함수가 호출될 때의 assembly를 분석해보자.
```cpp
void Func(int a, int b)
{
    int c = a + b;
}
```
이 함수는 MSVC, x86 플랫폼으로 빌드되었기 때문에 cdecl 함수호출규약을 사용할 것이다.  
cdecl 함수호출규약은 함수 파라미터를 가장 오른쪽것부터 가장 왼쪽것까지(RTL) 차례대로 stack에 입력한다(일부 파라미터는 레지스터에 저장되기도 한다).  
그리고 stack 메모리영역 정리 작업을 함수를 호출한 코드가 수행한다.

assembly
```cpp
    Func(10, 20);
0x00ED1270  push        14h                  // stack에 20 push (가장 오른쪽것부터 stack에 push)
0x00ED1272  push        0Ah                  // stack에 10 push
0x00ED1274  call        Func (0ED1110h)      // stack에 파라미터를 push 한 다음 함수를 call 한다.

// 함수 호출됨
void Func(int a, int b)
{
0x00ED1110  push        ebp
0x00ED1111  mov         ebp,esp
0x00ED1113  push        ecx
    int c = a + b;
0x00ED1114  mov         eax,dword ptr [ebp+8]      // a 변수의 값은 ebp 보다 아래에 있다. (함수 파라미터는 ebp보다 아래쪽에 위치한다)
0x00ED1117  add         eax,dword ptr [ebp+0Ch]    // b 변수의 값은 ebp 보다 아래에 있다.
0x00ED111A  mov         dword ptr [ebp-4],eax      // c 변수의 값은 ebp 보다 위에 있다. (지역변수는 ebp보다 위쪽에 위치한다)
}
0x00ED111D  mov         esp,ebp
0x00ED111F  pop         ebp
0x00ED1120  ret                    // 함수 종료

0x00ED1279  add         esp,8                  // 함수가 종료된 뒤, 함수 호출자가 esp를 증가시켜 Func 함수의 stack 메모리를 해제한다.
```


## 2. cdecl 함수 호출 분석
아래 cdecl 함수가 호출될 때의 assembly를 분석해보자.
```cpp
int __cdecl Test_cdecl(int a, int b, int c, int d, int e, int f, int g)
{
    return a + b + c + d + e + f + g;
}
```

assembly
```cpp
    // 함수 호출
    Test_cdecl(a1, a2, 30, 40, 50, 60, 70);
0x00ED128A  push        46h                           // stack에 70 push
0x00ED128C  push        3Ch                           // stack에 60 push
0x00ED128E  push        32h                           // stack에 50 push
0x00ED1290  push        28h                           // stack에 40 push
0x00ED1292  push        1Eh                           // stack에 30 push
0x00ED1294  mov         eax,dword ptr [a2]            // eax에 a2 값을 넣음. (일부 파라미터는 stack에 저장하고, 일부 파라미터는 레지스터에 저장한다)
0x00ED1297  push        eax                           // stack에 eax push
0x00ED1298  mov         ecx,dword ptr [a1]            // ecx에 a1 값을 넣음
0x00ED129B  push        ecx                           // stack에 ecx push
0x00ED129C  call        Test_cdecl (0ED1150h)         // Test_cdecl 함수 call

// 함수 호출됨
int __cdecl Test_cdecl(int a, int b, int c, int d, int e, int f, int g)
{
0x00ED1150  push        ebp
0x00ED1151  mov         ebp,esp
    return a + b + c + d + e + f + g;
0x00ED1153  mov         eax,dword ptr [ebp + 8]
0x00ED1156  add         eax,dword ptr [ebp + 0Ch]
0x00ED1159  add         eax,dword ptr [ebp + 10h]
0x00ED115C  add         eax,dword ptr [ebp + 14h]
0x00ED115F  add         eax,dword ptr [ebp + 18h]
0x00ED1162  add         eax,dword ptr [ebp + 1Ch]
0x00ED1165  add         eax,dword ptr [ebp + 20h]
}
0x00ED1168  pop         ebp
0x00ED1169  ret                                      // return

0x00ED12A1  add         esp,1Ch                      // 함수가 종료된 뒤 esp+28 하여 파라미터 전달에 사용된 stack 메모리를 해제함.
```


## 3. stdcall 함수 호출 분석
아래 stdcall 함수가 호출될 때의 assembly를 분석해보자.  
stdcall 함수호출규약은 함수 내부에서 stack 메모리를 해제한다.
```cpp
int __stdcall Test_stdcall(int a, int b, int c, int d, int e, int f, int g)
{
    return a + b + c + d + e + f + g;
}
```

assembly
```cpp
    // 함수 호출
    Test_stdcall(a1, a2, 30, 40, 50, 60, 70);
0x00ED12A4  push        46h                           // stack에 70 push
0x00ED12A6  push        3Ch                           // stack에 60 push
0x00ED12A8  push        32h                           // stack에 50 push
0x00ED12AA  push        28h                           // stack에 40 push
0x00ED12AC  push        1Eh                           // stack에 30 push
0x00ED12AE  mov         edx,dword ptr [a2]            // edx에 a2 값을 넣음. (일부 파라미터는 stack에 저장하고, 일부 파라미터는 레지스터에 저장한다)
0x00ED12B1  push        edx                           // stack에 edx push
0x00ED12B2  mov         eax,dword ptr [a1]            // eax에 a1 값을 넣음
0x00ED12B5  push        eax                           // stack에 eax push
0x00ED12B6  call        Test_stdcall (0ED1130h)       // Test_stdcall 함수 call

// 함수 호출됨
int __stdcall Test_stdcall(int a, int b, int c, int d, int e, int f, int g)
{
0x00ED1130  push        ebp
0x00ED1131  mov         ebp,esp
    return a + b + c + d + e + f + g;
0x00ED1133  mov         eax,dword ptr [ebp + 8]
0x00ED1136  add         eax,dword ptr [ebp + 0Ch]
0x00ED1139  add         eax,dword ptr [ebp + 10h]
0x00ED113C  add         eax,dword ptr [ebp + 14h]
0x00ED113F  add         eax,dword ptr [ebp + 18h]
0x00ED1142  add         eax,dword ptr [ebp + 1Ch]
0x00ED1145  add         eax,dword ptr [ebp + 20h]
}
0x00ED1148  pop         ebp
0x00ED1149  ret         1Ch                           // 함수를 종료하며 stack 에서 28bytes 만큼의 메모리를 해제한다.
```


### 4. cdecl 과 stdcall
x86 플랫폼에서의 함수호출규약 기본값은 `cdecl` 이다. (stack 메모리를 함수 호출자가 정리)  
그리고 x64 플랫폼에서는 Windows의 경우에는 `Microsoft x64 Calling Convention`을 사용하고, Linux에서는 `System V AMD64 ABI` 를 사용하는데, 둘다 stack 메모리를 함수 호출자가 정리한다.  
그래서 stack 메모리는 거의 항상 함수 호출자가 정리한다고 봐도 될듯하다.  
`stdcall`을 일부러 사용할 이유는 없어보인다.


### 5. 구조체를 함수 파라미터로 전달
아래 함수는 Data 구조체를 파라미터로 전달받는다.
```cpp
struct Data
{
    int a;
    double d;
    float e;
    __int64 f;
    char g[100];
};

int funcRequireStruct(Data d)
{
    return (int)d.a + (int)d.d + (int)d.e + (int)d.f + (int)d.g[99];
}
```

구조체 전달 코드 분석
assembly
```cpp
    // 구조체 선언
    Data stData = { 0, };
    stData.a = 1;
    stData.d = 2.0;
    stData.e = 3.f;
    stData.f = 4;
    stData.g[99] = 100;

    // 함수 호출됨
    r = funcRequireStruct(stData);
0x00571382  sub         esp,88h                             // stack 영역 136 바이트 확보(Data 구조체 크기)
0x00571388  mov         ecx,22h                             // ecx에 34(34*4=136) 입력
0x0057138D  lea         esi,[stData]                        // esi에 stData 주소 입력
0x00571393  mov         edi,esp                             // edi에 esp 입력
0x00571395  rep movs    dword ptr es:[edi],dword ptr [esi]  // ecx 값 만큼(34) 반복하여 [esi] 위치의 값을 4byte(dword)씩 [edi]로 옮김. 즉, stack 영역에 구조체를 memcpy 한다.
0x00571397  call        funcRequireStruct (0571200h)        // 함수 호출
0x0057139C  add         esp,88h                             // stack 메모리 해제
0x005713A2  mov         dword ptr [r],eax  
```
구조체를 값으로 전달할 때는, stack 영역에 전달할 구조체를 복사한 다음 함수를 호출하고, 함수에서는 호출 전 복사된 값을 사용한다.
