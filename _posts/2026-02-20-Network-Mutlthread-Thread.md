---
title: 스레드
date: 2026-02-20 17:05:21 +0900
categories: [멀티스레드, Thread]
tags: [network, thread]
description: 스레드
---

## 프로세스와 스레드

프로세스(process)는 1개 프로세스 내의 스레드(thread)들을 관리하는 껍데기 이다.  
프로세스가 가지고 있는것:
- 자신만의 가상 메모리 공간
- Code, Data, Heap 영역 등의 메모리 공간
- 핸들 테이블
- 스레드 핸들, 파일 핸들, 소켓, 세마포어 등의 운영체제가 할당한 시스템 자원

실질적인 코드의 수행은 스레드 단위로 이루어진다.  
스레드가 가지고 있는것:
- 자신만의 스택 메모리 영역
- 코드 실행과 연산을 위한 레지스터 세트(Register Set)
- 스레드 컨텍스트(ETHREAD block, TEB, CONTEXT 구조체 등)
  - '스레드 컨텍스트' 라고 하는것은 스레드 1개를 설명할 수 있는 정보를 말한다.  

## CONTEXT 구조체
CONTEXT 구조체는 CPU core에서 실행중인 스레드의 현재 상태를 스냅샷으로 저장한 구조체이다.  
주로 레지스터 값들이 저장되어 있다.  

스레드의 CONTEXT 출력해보기
```cpp
#include <windows.h>
#include <thread>
#include <format>
#include <iostream>
#include <string>

// 스레드 컨텍스트 출력 함수
void PrintThreadContext( HANDLE hThread )
{
    // 스레드 일시정지. 스레드 컨텍스트는 스레드 실행중에는 실시간으로 값이 변경되기 때문에 반드시 스레드를 정지시킨다음 확인해야 한다.
    SuspendThread( hThread );

    // CONTEXT 구조체 선언 및 초기화
    CONTEXT ctx;
    ZeroMemory( &ctx, sizeof( CONTEXT ) );

    // 가져올 정보의 범위 지정
    ctx.ContextFlags = CONTEXT_ALL;

    // 스레드 컨텍스트 얻기
    if ( GetThreadContext( hThread, &ctx ) )
    {
        // 스레드 컨텍스트 출력
        std::string strContext;

        strContext += std::format( "Rsp = {:#x}\n", ctx.Rsp );
        strContext += std::format( "Rip = {:#x}\n", ctx.Rip );
        strContext += std::format( "SegSs = {:#x}\n", ctx.SegSs );
        strContext += std::format( "SegCs = {:#x}\n", ctx.SegCs );
        strContext += std::format( "EFlags = {:#x}\n", ctx.EFlags );

        strContext += std::format( "Rax = {:#x}\n", ctx.Rax );
        strContext += std::format( "Rcx = {:#x}\n", ctx.Rcx );
        strContext += std::format( "Rdx = {:#x}\n", ctx.Rdx );
        strContext += std::format( "Rbx = {:#x}\n", ctx.Rbx );
        strContext += std::format( "Rbp = {:#x}\n", ctx.Rbp );
        strContext += std::format( "Rsi = {:#x}\n", ctx.Rsi );
        strContext += std::format( "Rdi = {:#x}\n", ctx.Rdi );

        strContext += std::format( "SegDs = {:#x}\n", ctx.SegDs );
        strContext += std::format( "SegEs = {:#x}\n", ctx.SegEs );
        strContext += std::format( "SegFs = {:#x}\n", ctx.SegFs );
        strContext += std::format( "SegGs = {:#x}\n", ctx.SegGs );
        strContext += std::format( "Rdi = {:#x}\n", ctx.Rdi );

        std::cout << strContext << std::endl;
    }
    else
    {
        std::cerr << "Error: " << GetLastError() << std::endl;
    }

    // 스레드 일시정지 해제
    ResumeThread( hThread );
}

int main() 
{

    // 스레드 생성
    std::thread t = std::thread( []()
    {
        int sum = 0;
        for ( int i = 0; i < 10000000; ++i )
        {
            sum += i;
        }
        std::cout << sum << std::endl;
    } );
    
    // 스레드 핸들 얻기
    HANDLE hThread = (HANDLE)t.native_handle();

    // 스레드 컨텍스트 출력
    PrintThreadContext( hThread );

    t.join();

    return 0;
}
```
출력결과:  
![CONTEXT](/assets/img/posts/2026-02-20-Network-Mutlthread-Thread-img1.png)

## Thread Environment Block(TEB)
TEB는 스레드가 실행되는 동안 유저영역에서 필요한 정보를 저장하고 있는 구조체이다.  
TEB는 프로세스의 유저영역 메모리공간에 존재한다.  

여기에는 아래와 같은 정보들이 들어있다:
- 스레드 ID
- 시스템 오류코드(GetLastError 함수로 얻는값)
- TLS(Thread Local Storage) 영역의 주소
- Stack 메모리영역 시작주소
- PEB(Process Environment Block) 주소
- 기타 등등

스레드의 TEB 데이터 주소는 GS 레지스터에 입력되어 있다.

스레드의 TEB 출력해보기
```cpp
#include <iostream>
#include <windows.h>
#include <intrin.h>   // __readgsqword 함수 include
#include <format>

int main()
{
    // 64비트 Windows 환경에서 TEB 내의 주요 필드 오프셋:
    // 0x00: StackBase
    // 0x08: StackLimit
    // 0x30: TEB 자기 자신을 가리키는 포인터
    // 0x60: PEB (Process Environment Block) 주소
    // 0x68: GetLastError() 값이 저장되는 위치

    // GS 레지스터의 0x30 오프셋에서 TEB 주소를 얻는다
    unsigned char* pTEB = (unsigned char*)__readgsqword( 0x30 );
    std::cout << std::format( "TEB 주소 : 0x{:p}", (void*)pTEB ) << std::endl;

    // 시스템 오류코드(GetLastError() 값) 출력
    DWORD* pLastError = (DWORD*)( pTEB + 0x68 );
    std::cout << std::format( "LastError : {}\n", *pLastError );

    // PEB(Process Environment Block) 주소 출력
    void** pPEB = (void**)( pTEB + 0x60 );
    std::cout << std::format( "PEB 주소 : 0x{:p}\n", *pPEB );

    // TIB(Thread Information Block) 구조체를 얻고 내용 출력
    PNT_TIB pTIB = (PNT_TIB)pTEB;
    std::cout << std::format( "Stack Base : 0x{:p}\n", pTIB->StackBase );
    std::cout << std::format( "Stack Limit : 0x{:p}\n", pTIB->StackLimit );

    return 0;
}
```
출력결과:  
![TEB](/assets/img/posts/2026-02-20-Network-Mutlthread-Thread-img2.png)



## ETHREAD(Executive Thread) Block
ETHREAD Block은 커널이 스레드 스케줄링 등을 위해 관리하는 구조체이다.  
ETHREAD Block은 커널영역 메모리공간에 존재한다.  
ETHREAD Block은 커널영역에 있어서 일반적으로는 정보를 조회할 수 없다.  

ETHREAD Block에는 아래와 같은 정보들이 있다:
- Thread time                  : 스레드 생성과 종료시간 정보
- Start address                : 스레드 시작 함수 주소
- ALPC information             : 스레드가 기다리는 Message ID 그리고 message 주소
- I/O information              : 대기중인 I/O request packets (IRPs)
- KTHREAD block                : KTHREAD block은 TCB, thread control block 으로도 불린다. 상세 내용은 아래 참조

KTHREAD block에는 아래와 같은 정보들이 있다:
- Execution time               : 유저, 커널 총 CPU 소요시간
- Cycle time                   : 총 CPU cycle 시간
- Scheduling information       : 스케줄링을 위한 정보. 스레드 할당시간, affinity mask, 스케줄링 상태 등
- Wait information             : 스레드가 기다리는 오브젝트 리스트, 기다리는 이유, wait 했을때의 IRQL, wait 결과, 스레드가 wait 상태가된 시간
- APC queues                   : 대기중인 유저모드 및 커널모드 APCs, alerted flag, APC 비활성화 플래그
- Pointer to TEB               : TEB 구조체 주소


