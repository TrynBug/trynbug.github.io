---
title: C++ 함수 Assembly 분석
date: 2026-01-30 00:00:00 +0900
categories: [C++, Assembly]
tags: [c++, function, assembly]
description: C++ 함수 Assembly 분석
---

## 요약
- 함수가 호출되면 이전 함수의 ebp 값을 stack에 저장해둔다.
  - 때문에 함수가 stack 메모리영역을 잘못 건드리면 이전 함수의 ebp 값이 깨질 수 있다.
- 함수가 호출되면 함수 내에서 사용할 지역변수 메모리영역을 stack에서 한번에 확보한다.
- 함수의 지역변수는 ebp 주소를 기준으로 몇 번째 위치에 있느냐를 기준으로 식별된다.
- 함수의 리턴값은 eax 레지스터에 저장된다.
- 함수가 종료되었을 때 돌아갈 위치는 eip 레지스터에 저장된다.
  - 함수가 호출될 때 eip 레지스터값도 stack 메모리영역에 백업되는데, 이때문에 함수가 stack 메모리영역을 잘못 건드리면 eip 값이 깨질 수 있다.


## 1. 함수 내부 Assembly 분석

아래 함수의 assembly를 분석해보자
```cpp
void Func()
{
    int a;
    a = 0;
    int b = 0;
    int c = 1;
}
```

#### Func() 함수의 assembly
참고사항:  
- esp(extended stack pointer) 레지스터 : 사용중인 stack 메모리영역의 최상단 주소(가장 최근에 사용한 위치)를 가르킨다.
- ebp(extended base pointer) 레지스터 : 현재 실행중인 함수의 stack 메모리영역 시작주소를 가르킨다.

```cpp
// Func 함수를 호출하고 assembly 명령어를 살펴본다.
void Func()
{
  // 여기서 0x006B1030 는 코드의 주소, push는 명령어, ebp는 파라미터를 나타낸다.

  // 이전 함수의 ebp 레지스터값을 stack 메모리영역에 push하여 이전 함수의 ebp를 저장해둔다.
0x006B1030  push        ebp          
  // esp 레지스터값을 ebp에 옮겨서 현재 함수의 stack 영역 시작주소를 확보한다.
0x006B1031  mov         ebp,esp  
  // 아래 assembly 코드를 보면 esp에서 12를 뺌으로써 12byte의 stack 메모리 공간을 확보한다.
  // 즉, 함수가 호출되면 함수내의 모든 지역변수에 필요한 메모리 공간을 한번에 확보한다. 
  // 지역변수 라는것은 ebp로부터 몇 번째 위치에 있는가로 식별되는 것이다.
  // 그래서 지역변수가 많다고 해서 함수 호출에 시간이 더 걸리거나 하지는 않는다.
0x006B1033  sub         esp,0Ch  

  // a 변수 선언에 대한 어셈블리 명령어는 없다. 왜냐하면 지역변수는 ebp를 기준으로 몇 번째 위치에 있냐로 식별되기 때문이다.
    int a;
    a = 0;
  // a 변수는 ebp를 기준으로 -4 위치의 메모리에 존재하는것을 알수있다.
0x006B1036  mov         dword ptr [ebp-4],0  
    int b = 0;
0x006B103D  mov         dword ptr [ebp-8],0  
    int c = 1;
0x006B1044  mov         dword ptr [ebp-0Ch],1  
}
  // (Debug 구성으로 빌드했을 때) 함수 종료 부분에서는 ebp 레지스터 값이 올바른지 체크한다.
  // 여기서 ebp값을 검사하는 이유는 함수를 잘못 작성하여 함수가 ebp 레지스터 값을 실수로 변경할 수가 있기 때문이다. 그렇게 되면 프로그램이 제대로 작동하지 않는다.
  // ebp 값을 esp와 비교하여 올바른지 체크하는데, ebp 값은 stack에 저장해두었기 때문에 함수가 실수로 변경할 수 있지만 esp 값은 레지스터에만 존재하기 때문에 함수가 변경할 수 없어서 esp값을 기준으로 ebp가 올바른지 확인하는 것이다.
0x006B104B  add         esp,0Ch
0x006B104D  cmp         ebp,esp
0x006B104E  call        __RTC_CheckEsp (0F61235h)  // ebp != esp 일때만 호출되는 함수. 프로그램 크래시를 발생시킨다. (Debug 구성으로 빌드했을 때만 삽입되는 코드이다)
  // ebp 값을 esp에 입력한다. 즉, esp 값을 현재함수의 stack영역 시작주소로 변경한다.
0x006B1052  mov         esp,ebp  
  // stack 영역 시작주소(=이전 함수의 ebp 레지스터값을 백업한위치) 값을 pop하여 ebp 레지스터에 저장한다. 즉, ebp 레지스터 값을 이전함수의 ebp 값으로 복구한다.
0x006B1056  pop         ebp  
  // 함수 return
0x006B105a  ret  
```


## 2. 함수가 ebp 값을 실수로 변경할 경우 발생하는 일
함수가 ebp 값을 실수로 변경하면 어떻게 될까?

아래의 Fail 함수는 Base 함수의 ebp 값을 실수로 변경한다.

```cpp
void Fail()
{
    int a = 111;
    int* p = &a;
    *(p + 2) = 0;    // 스택 영역에서 a 변수 주소에서 8바이트 아래 위치(== 이전 함수의 ebp 값이 백업된 위치)의 값을 0으로 변경함.
}

void Base()
{
    int a = 1;
    int b = 2;

    Fail(); // Fail 함수가 리턴되면 Base 함수의 ebp가 0 이기 때문에 아래 코드에 오류가 발생한다.
    a = b;
}
```

assembly 분석
```cpp
// Fail 함수 호출됨
void Fail()
{
0x003C1050  push        ebp        // ebp를 stack에 저장해둔다. (이전 함수의 ebp를 백업)
0x003C1051  mov         ebp,esp    // ebp 값을 esp 값으로 변경한다.
0x003C1053  sub         esp,0Ch    // esp에 12를 빼서 stack 메모리를 확보한다.
    int a = 111;
0x003C1060  mov         dword ptr [ebp-4],6Fh  
    int* p = &a;
0x003C1067  lea         eax,[ebp-4]  
0x003C106A  mov         dword ptr [ebp-8],eax  
    *(p + 2) = 0;    // 스택 영역에서 a 변수 주소에서 8바이트 아래 위치(== 이전 함수의 ebp 값이 백업된 위치)의 값을 0으로 변경함.
0x003C106D  mov         ecx,dword ptr [ebp-8]  
0x003C1070  mov         dword ptr [ecx+8],0  
}
0x003C1074  mov         esp,ebp  
0x003C1078  pop         ebp
0x003C107C  ret    // 함수 return


// Fail 함수가 return된 뒤의 Base 함수 코드
void Base()
{
    int a = 1;
    int b = 2;

    Fail();
0x00A610B2  call        Fail (0A61050h)  
    a = b;
0x00A610B7  mov         eax,dword ptr [ebp-4]   // b의 위치는 ebp-4 인데, 이 때의 ebp 값을 조회해보면 값이 0이다. 때문에 b변수(ebp-4)에 접근하는 순간 메모리 액세스 위반 오류가 발생한다.
0x00A610BA  mov         dword ptr [ebp-8],eax  
}
```


## 3. 함수의 리턴값은 eax 레지스터에 저장된다.

함수의 리턴값은 eax(extended accumulator register) 레지스터에 저장되어 전달된다.  
int를 리턴하는 함수를 호출하는 코드를 분석해보자.
```cpp
int Test_return()
{
    int x = 0;
    x++;
    return x;
}

int main()
{
    int r = Test_return();
}
```

assembly
```cpp
    int r = Test_return();
00C51307  call        Test_return (0C511B0h)         // Test_return 함수 call

int Test_return()
{
00C511B0  push        ebp
00C511B1  mov         ebp,esp
00C511B3  push        ecx
    int x = 0;
00C511B4  mov         dword ptr [x],0
    x++;
00C511BB  mov         eax,dword ptr [x]
00C511BE  add         eax,1
00C511C1  mov         dword ptr [x],eax
    return x;
00C511C4  mov         eax,dword ptr [x]              // return 할 값인 x 변수 값을 eax에 넣음
}
00C511C7  mov         esp,ebp
00C511C9  pop         ebp
00C511CA  ret

00C5130C  mov         dword ptr [r],eax              // eax 값을 r 변수에 넣음
```



## 4. 함수가 종료되었을 때 돌아갈 위치는 eip 레지스터에 저장된다.

eip(extended instruction pointer) 레지스터는 다음번 실행할 코드의 주소를 저장하고 있다.  
함수를 호출할 때는 eip 레지스터 값을 호출할 함수의 코드 주소로 변경하면 된다. 그러면 다음번에 해당 함수의 코드가 실행된다.  
그런데 함수가 종료된 다음 돌아갈 위치는 어떻게 알수있을까?  
아래 함수 호출 코드를 분석해보자.  
```cpp
void Test()
{
    int x = 0;
}

int main()
{
    Test();
}
```

assembly
```cpp
int main()
{
// 현재 esp 레지스터 값 = 0x00effed4
// 현재 eip 레지스터 값 = 0x00871044

// Test 함수를 호출한다.
	Test();
00871044  call        Test (0871000h)
// call 명령어는 함수가 종료된 다음 돌아갈 주소(현재 eip를 기준으로 다음 명령어 주소)를 stack에 백업한 다음, eip 값을 Test 함수 시작주소로 변경한다.
// eip 값을 stack에 저장했기 때문에 esp 값은 -4 된다.
// 현재 esp 레지스터 값 = 0x00effed0 (-4 됨)
// 현재 eip 레지스터 값 = 0x00871000 (Test 함수의 시작주소)


void Test()
{
00871000  push        ebp
00871001  mov         ebp,esp
00871003  push        ecx
	int x = 0;
00871004  mov         dword ptr [x],0
}
0087100B  mov         esp,ebp
0087100D  pop         ebp
0087100E  ret                  // 함수 종료
// ret 명령어는 현재 esp 주소위치의 값(백업해둔 eip값)을 pop 하여 eip에 복원하고 esp를 +4한다.
// 즉, 함수가 호출된 위치로 돌아간다.
// 현재 esp 레지스터 값 = 0x00effed4
// 현재 eip 레지스터 값 = 0x00871049 (함수 호출 다음 코드의 주소)
```
