---
title: 가상 메모리
date: 2026-02-17 23:28:12 +0900
categories: [OS, 메모리]
tags: [OS, 메모리]
description: 가상 메모리
---

## 1. 가상 메모리(virtual memory)

가상 메모리 공간은 프로세스별로 존재한다. 그리고 프로세스간에 완전히 독립적이다.  
때문에 하나의 프로세스가 다른 프로세스의 가상메모리 공간에 간섭할 수 없다.  

가상 메모리는 4KB 단위의 page로 나뉘어져 있다.  
물리 메모리도 4KB 단위의 frame으로 나뉘어져 있다.  
물리 메모리가 page-out 되어 디스크에 저장되었을 때 이 데이터도 4KB 단위의 sector로 관리된다.  
![Buffer](/assets/img/posts/2026-02-17-OS-virtual-memory-img.png)

가상 메모리의 page는 3가지의 상태가 있다.
- free
  - 프로세스가 아직 사용하지 않은 page. 접근하면 액세스 위반(Access Violation) 예외가 발생한다.
- reserved
  - 사용 예약한 page. 예약 한다는것은 가상메모리공간을 확보한다는 것이다. 이 상태에서는 물리메모리와 아무런 관련이 없다.
- committed (committed는 아래 2가지 상태로 또 나눌 수 있다.)
  - committed 상태이지만 아직 실제로 접근은 안한 상황
    -  물리 메모리에는 매핑되지 않았지만 디스크의 스왑파일(페이징 파일)에 공간을 확보해둔다.
  - committed 상태이고 실제로 접근한 상황
    - 물리 메모리에 실제로 매핑된다.

free 상태의 가상메모리는 물리메모리에 매핑되지 않는다.  
reserved 상태의 가상메모리는 물리메모리에 매핑되지 않는다.  
committed 상태의 가상메모리는 실제로 접근하기 전까지는 물리메모리에 매핑되지 않는다.  
committed 상태의 가상메모리와 매핑된 물리메모리는 오랫동안 사용하지 않으면 page-out 되어 하드디스크에 저장된다.  


## Windows API로 가상메모리 할당, 해제하기

먼저 시스템의 가상메모리 할당 단위를 알아보자.
```cpp
int main()
{
    // 아래 함수로 가상메모리의 할당단위를 알아낼 수 있다.
    SYSTEM_INFO sys;
    GetSystemInfo(&sys);

    // 현재 sys.dwAllocationGranularity 값은 65536 이다. 즉, 가상메모리 할당 단위는 64KB 이다.
    // 가상메모리 할당 단위가 64KB 이면 해제 단위도 64KB 이다.
    // 그리고 할당한 가상메모리 내에서 page(4KB) 단위로 메모리의 상태(free, reserved, committed)를 관리할 수 있다.
    std::cout << "가상메모리 할당단위: " << sys.dwAllocationGranularity << std::endl;

  return 0;
}
```

가상메모리는 VirtualAlloc(주소, 크기, 상태, 속성) 함수로 할당할 수 있다.  
각각의 파라미터를 알아보자.  

1. 주소
  - 주소에 NULL을 입력하면 알아서 빈 자리를 찾아 할당한다. 그래서 이 때에는 reserved 없이 바로 committed 된다.
  - 메모리 주소를 입력하면 해당 주소 위치에 할당한다.
  - 주소를 지정하는 경우에는 최초 `MEM_RESERVE`는 64KB 메모리 경계에 맞춰주어야 한다. 왜냐하면 가상메모리 할당단위가 64KB 이기 때문이다.
  - 가상메모리를 최초 할당(여기서 할당은 예약 이후의 단계) 받으면 그 주소는 반드시 64KB 경계일 것이다.
  - 최초 할당 이후 추가로 할당할 때는 할당되는 주소는 page 크기인 4KB 경계일 것이다.

2. 크기
  - 무조건 4KB 단위로 지정된다. 값을 4KB 단위로 입력하지 않아도 4KB 단위로 인식된다.
  - `MEM_RESERVE` 할때 지정한 크기 이상에 대해선 크기 변경이 불가능하다.

3. 상태
  - 최초에는 `MEM_RESERVE` 또는 `MEM_RESERVE | MEM_COMMIT` 이 가능하며, 주소 미지정시 `MEM_COMMIT` 만 된다.
  - 주소가 지정된 경우는 `MEM_COMMIT` 만 단독 사용이 불가능하다.
  - `MEM_DECOMMIT` 을 하면 `MEM_RESERVE` 상태로 돌아온다.
  - `MEM_FREE` 를 하려면 64KB 영역 전체를 `MEM_FREE` 해야 하고, 특정 page만 `MEM_FREE` 하는것은 불가능하다.

4. 속성
  - `PAGE_READONLY` : 읽기전용
  - `PAGE_NOACCESS` : 액세스 금지
  - `PAGE_READWRITE` : 읽기쓰기. 일반적으로 사용된다.
  - guard 속성을 사용자가 지정할 수는 없다.
  - 페이지 단위로 속성은 수시로 변경이 가능하다.


그럼이제 가상메모리를 할당하고 해제해보자.
```cpp
int main()
{
    // 아래와 같이 주소를 지정하는 경우에는 MEM_COMMIT 단독 사용은 불가능하다. MEM_COMMIT 단독 사용하면 NULL이 리턴된다.
    // 간혹 이런 코드가 작동이 되는경우가 있을 수 있다.
    // 그건 우연하게 해당 주소가 reserved 또는 committed 되어 있던 경우이다.
    char* p = (char*)VirtualAlloc((VOID*)0x00a00010, 4096 * 4, MEM_COMMIT, PAGE_READWRITE);

    // 주소파라미터에 NULL을 입력하면 미사용중인 공간 두 페이지를 예약 + 커밋 한다.
    // 이 경우 64KB 경계의 미사용중인 공간을 자동으로 찾아준다.
    // 리턴되는 주소는 64KB 단위이며, 이후 공간을 추가로 확장할 수 없다.
    // 그리고 4096 * 2 크기만 할당되었기 때문에 나머지 4096 * 14 만큼의 가상메모리 영역은 사용할 수 없게 된다.(예약도 안됨)
    // MEM_COMMIT 을 하면 해당 페이지의 메모리는 0으로 초기화된다.
    // MEM_COMMIT 된 페이지를 다시 MEM_COMMIT 하는 경우에는 초기화되지 않는다.
    p = (char*)VirtualAlloc(NULL, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);

    // 가상메모리 공간을 추가로 MEM_COMMIT 하는 아래 함수는 실패한다. 
    // 왜냐하면 가상메모리 공간의 8KB만 할당받아서 그 뒤쪽의 공간은 사용할 수 없게 됐기 때문이다.
    char* pRet = (char*)VirtualAlloc(p + 4096 * 2, 4096, MEM_COMMIT, PAGE_READWRITE);
    // 가상메모리 공간을 추가로 MEM_RESERVE 하는 이 함수도 실패한다. 이유는 위와 동일하다.
    pRet = (char*)VirtualAlloc(p + 4096 * 2, 4096, MEM_RESERVE, PAGE_READWRITE);
    // 가상메모리 공간에서 예약크기를 변경하려는 이 함수도 실패한다. 이유는 위와 동일하다. 
    pRet = (char*)VirtualAlloc(p, 4096 * 4, MEM_RESERVE, PAGE_READWRITE);
    

    // 앞에서 2페이지를 예약+커밋 한거 중에서 뒤쪽 1페이지를 MEM_DECOMMIT 한다.  뒤의 1페이지 디커밋 -> 다시 예약
    // 페이지가 MEM_DECOMMIT 되면 메모리뷰에서 봤을 때 모두 ?? 로 보인다.
    bool bResult = VirtualFree(p + 4096, 4096, MEM_DECOMMIT);
    // MEM_DECOMMIT 한 페이지를 다시 MEM_COMMIT 한다.
    pRet = (char*)VirtualAlloc(p + 4096, 4096, MEM_COMMIT, PAGE_READWRITE);

    // 최초 할당받은 가상메모리 주소가 아닌 위치를 MEM_RELEASE 하려고 한다. 이 함수는 실패한다.
    bResult = VirtualFree(p + 4096, 4096, MEM_RELEASE);

    // 최초 할당받은 가상메모리 주소를 MEM_RELEASE 하려고 한다.
    // 이 함수는 실패하는데 이유는 크기 파라미터에 값이 들어갔기 때문이다.
    // MEM_RELEASE 를 할 때는 크기 파라미터에 값이 들어가면 안된다.
    // 그리고 무조건 64KB 경계의 주소만을 입력해야 한다.
    bResult = VirtualFree(p, 4096, MEM_RELEASE);

    // 아래 MEM_RELEASE 함수는 성공한다.
    bResult = VirtualFree(p, 0, MEM_RELEASE);
    


    // 4페이지 예약. 성공
    p = (char*)VirtualAlloc(NULL, 4096 * 4, MEM_RESERVE, PAGE_READWRITE);
    // 처음 2페이지 커밋. 성공
    char* p1 = (char*)VirtualAlloc(p, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
    // 그뒤의 2페이지 커밋. 성공
    char* p2 = (char*)VirtualAlloc(p + 4096 * 2, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
    // 그뒤의 2페이지 커밋. 실패. 
    // 실패이유는 처음에 4페이지만 예약했기 때문이다.
    char* p3 = (char*)VirtualAlloc(p + 4096 * 4, 4096 * 2, MEM_COMMIT, PAGE_READWRITE);
    // 4페이지 예약된 가상메모리 주소를 10페이지 예약으로 변경하려 한다. 실패.
    // 실패이유는 처음에 4페이지만 예약했기 때문이다.
    char* pInvalid = (char*)VirtualAlloc(NULL, 4096 * 10, MEM_RESERVE, PAGE_READWRITE);
    

    // 커밋된 4개 페이지의 모든 메모리를 접근해본다.
    // 이렇게 하면 실제로 물리메모리 매핑되고, 작업관리자에서 commit 크기를 보면 값이 증가하는것을 볼 수 있다.
    int a = 0;
    for (int iCnt = 0; iCnt < 4096 * 4; iCnt++)
    {
        a = *p;
        p++;
    }

    // 앞에서 4페이지 예약한것 중에 4번째 페이지의 속성을 PAGE_NOACCESS 로 변경한다.
    p3 = (char*)VirtualAlloc(p + 4096 * 3, 4096, MEM_COMMIT, PAGE_NOACCESS);
    
    // 4번째 페이지에 접근해본다.
    for (int iCnt = 0; iCnt < 4096 * 4; iCnt++)
    {
        a = *p; // 4번째 페이지에 접근할 때 액세스 위반 오류가 발생한다.
        p++;
    }

    // 앞에서 4페이지 예약한것 중에 4번째 페이지의 속성을 PAGE_READONLY 로 변경한다. 읽기는 되지만 쓰기는 안된다.
    p3 = (char*)VirtualAlloc(p + 4096 * 3, 4096, MEM_COMMIT, PAGE_READONLY);
    
    // 4번째 페이지에서 읽기만 하는것은 상관없다.
    for (int iCnt = 0; iCnt < 4096 * 4; iCnt++)
    {
        a = *p;
        p++;
    }

    return 0;
}
```
