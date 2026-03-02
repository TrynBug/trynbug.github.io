---
title: 가상메모리를 물리메모리로 변환하기
date: 2026-03-02 01:02:45 +0900
categories: [OS, 메모리]
tags: [OS, 메모리]
description: 가상메모리 주소를 물리메모리 주소로 변환하기
---

## 1. 가상메모리의 범위
일반적인 x64 아키텍처의 OS는 48bit 가상메모리 주소 체계를 사용한다.  
전체 8bytes 가상메모리 공간에서 사용가능한 영역의 범위를 48bit = 256TB 로 제한한다.  

| 파티션           | 가상메모리 주소                           | 크기  |
| ---------------- | ----------------------------------------- | ----- |
| NULL 포인터 할당 | 0x00000000'00000000 ~ 0x00000000'0000FFFF | 4KB   |
| 유저 영역        | 0x00000000'00010000 ~ 0x00007FFF'FFFFFFFF | 128TB |
| 접근금지 영역    | 0x00008000'00000000 ~ 0xFFFF7FFF'FFFFFFFF |       |
| 커널 영역        | 0xFFFF8000'00000000 ~ 0xFFFFFFFF'FFFFFFFF | 128TB |


## 2. 프로그램의 가상메모리 주소
프로그램을 빌드할 때 코드에서 사용하는 주소는 모두 가상메모리 주소이다.  
```cpp
  int a = 1;
00007FF631E81004  mov         dword ptr [0x000000d39b18f9f0],1  // 0x000000d39b18f9f0 주소 위치에 값 1을 입력한다.
```
여기서 0x000000d39b18f9f0 는 가상메모리 주소이다.  
그런데 실제 메모리는 물리메모리에 있다.  
프로세스는 CPU에게 0x000000d39b18f9f0 주소에 작업을 요청한다.  
그러면 CPU는 가상메모리 주소를 물리메모리 주소로 변환하여 사용해야 한다.  

## 3. 가상메모리 주소의 하위 12bit 변환
가상메모리 1개 page는 물리메모리 1개 frame에 매핑된다.  
그리고 1개 page는 4KB 이고, 1개 frame도 4KB 이다.  
그리고 4KB는 4096bytes 이다.  
4KB(=4096)는 12개 bit로 표현할 수 있다. 2진수로 1111 1111 1111 = 4095 이기 때문이다.  

그러면 어떤 가상메모리 page의 시작주소가 0x00a3b000 이라고 하자.  
그리고 가상메모리와 매핑된 물리메모리 frame의 시작주소가 0x01fe3000 이라고 하자.  
그러면 가상메모리 page 내에서 0xAAA 번째 위치의 주소(0x00a3baaa)는 물리메모리 frame 내에서도 0xAAA 번째 위치의 주소(0x01fe3aaa)일 것이다.  
즉, 가상메모리 주소를 물리메모리 주소로 변환할 때 가장 하위의 12bit는 변환하지 않아도 알 수 있다.  
![memory](/assets/img/posts/2026-03-02-OS-virtual-to-physical-img1.svg)

## 4. 가상메모리 주소의 상위 36bit 변환 : TLB
48bit 가상메모리 주소체계를 사용하기 때문에, 하위 12bit를 제외한 상위 36bit를 변환해야 한다.  
가상메모리 주소의 상위 36bit 변환은 TLB(Translation Lookaside Buffer) 라는 캐시메모리가 담당한다.  
TLB는 L1, L2, L3 캐시와는 다른 별도의 캐시메모리이다.  
TLB는 CPU의 MMU(Memory Management Unit)내에 존재한다.  
TLB에 들어있는 데이터는 key-value 구조이며, key(tag)는 가상메모리 page 번호, value(data)는 물리메모리 frame 번호이다.  
CPU가 TLB에서 tag에 해당하는 data를 찾을 때는 TLB의 레코드들에 병렬로 접근하여 한번에 찾는다.  

![memory](/assets/img/posts/2026-03-02-OS-virtual-to-physical-img2.svg)

TLB에 들어있는 1개 entry의 구조는 정확하지는 않지만 대략 아래와 같은 구조일 것이다.  

| 필드  | 설명                                                               |
| ----- | ------------------------------------------------------------------ |
| Tag   | 가상메모리 page 번호                                               |
| Data  | 물리메모리 frame 번호                                              |
| PCID  | 프로세스 식별자 (프로세스마다 독립적인 가상메모리 영역을 가지므로) |
| Flags | Valid flag, Dirty flag, Read/Write 권한 등                         |


## 5. 가상메모리 주소의 상위 36bit 변환 : Page Table
TLB에 처음부터 가상메모리-물리메모리 변환 정보가 들어있는 것은 아니다.  
TLB는 캐시메모리이기 때문에 '최근에 사용한 가상메모리-물리메모리 변환 정보'가 들어있다.  
그리고 TLB도 용량의 한계가 있기 때문에 오래 사용되지 않은 entry는 flush 한다.  
그러면 TLB에서 가상메모리-물리메모리 정보를 찾지 못했으면 다른곳에서 찾아야 한다.  
(TLB 에서 정보를 찾으면 TLB hit, 못찾으면 TLB miss 라고 한다.)  
이 때는 Page Table 에서 찾는다.  

Page Table은 프로세스마다 각각 존재한다. 프로세스마다 자신의 가상메모리 영역을 가지기 때문이다.  
Page Table은 메모리상에 존재하며, Page Table이 위치한 메모리 주소는 CPU의 CR3 레지스터에 입력되어 있다.  
Page Table에서 물리메모리 주소를 찾는 것을 page walk 라고 한다.  

Page Table에서 물리메모리 주소를 찾으려면 5개의 단계를 거친다(아래 그림 참조).  
1. 주소의 47~39번째 bit를 offset으로 하여 Page-Map Level-4(PML4) Table에서 entry를 찾는다.
2. 주소의 38~30번째 bit를 offset으로 하여 Page-Directory-Pointer(PDP) Table에서 entry를 찾는다.
3. 주소의 29~21번째 bit를 offset으로 하여 Page-Directory(PD) Table에서 entry를 찾는다.
4. 주소의 20~12번째 bit를 offset으로 하여 Page-Table(PT) 에서 entry를 찾는다. 여기에는 '물리메모리 frame의 시작주소'가 들어있다.
5. 주소의 11~00번째 bit는 물리메모리 frame 내에서의 offset 이다.
그래서 마지막 Page-Table(PT)에서 찾은 '물리메모리 frame의 시작주소 + 가상메모리주소의 11~00 번째 bit값'을 계산하면 물리메모리 주소를 얻을수있다.  

![memory](/assets/img/posts/2026-03-02-OS-virtual-to-physical-img3.png)
_https://connormcgarr.github.io/paging/_

Page Table에서 찾은 물리메모리 주소는 TLB에 없는 주소이고 방금 사용되었기 때문에 TLB에 캐싱된다.  
page walk 절차를 보면 알 수 있듯이 가벼운 작업은 아니다.  
그래서 TLB miss가 많이 발생하면 성능이 저하된다.  

## 6. Page Table에 찾는 데이터가 없을 때
가상메모리 주소를 물리메모리 주소로 변환하려고 TLB를 찾아봤는데, 원하는 정보를 찾지 못했다.  
그래서 Page Table에서 찾아봤는데, 여기에서도 찾지 못했다.  
이 경우는 해당 가상메모리주소는 가상메모리영역에 할당만 되어있고 아직 물리메모리에 매핑되지 않은 것이다.  
가상메모리를 할당만 하고 실제로 접근하지 않았다면 물리메모리와 매핑되지 않는다.  
그러면 운영체제는 해당 가상메모리 page를 물리메모리에 매핑시키고 Page Table을 업데이트 한다.  

## 7. 찾는 물리메모리가 page-out 되었을 때
물리메모리에서 오랫동안 사용되지 않은 물리메모리 frame은 운영체제에 의하여 디스크로 page-out될 수 있다.  
frame이 page-out 되었다면 Page Table에 해당 frame이 page-out 되었다는 정보가 기록된다.  
그러면 Page Table에는 물리메모리 주소가 저장되는것이 아니라 데이터가 swap 파일의 어느 위치에 저장되어 있는지가 기록된다.  

page walk를 통해 가상메모리주소에 해당하는 물리메모리주소를 찾으려 했는데, Page Table을 확인해봤더니 해당 frame은 page-out 되었다.  
그러면 CPU는 Page Fault 예외를 발생시키고, 운영체제가 이 예외를 받아서 처리한다.  
Page Fault 예외가 발생하면 운영체제는 요청된 데이터를 물리 메모리로 가져오고, Page Table을 업데이트하여 가상메모리주소-물리메모리주소 정보를 추가한다.  
그리고 CPU에게 올바른 물리메모리 주소를 반환한다.  

## 8. 가상메모리-물리메모리 주소 변환 순서도
![memory](/assets/img/posts/2026-03-02-OS-virtual-to-physical-img4.svg)
