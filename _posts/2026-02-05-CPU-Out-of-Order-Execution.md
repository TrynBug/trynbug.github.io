---
title: Out-of-Order-Execution
date: 2026-02-05 23:13:30 +0900
categories: [CPU, Out-of-Order-Execution]
tags: [cpu, outofordering]
description: Out-of-Order-Execution
---

## 1. Out-of-Order-Execution

참고: STORE = 쓰기, LOAD = 읽기

CPU는 성능을 높이기 위해 명령어를 순서대로 실행하지 않는다. 이것을 Out-of-Order Execution 이라고 한다.  
예를 들면 STORE 해야 하는 데이터들을 모아두었다가 메모리에 한번에 쓰거나,  
메모리에서 LOAD 해야하는 데이터들을 미리미리 읽어둔다.

STORE 해야하는 데이터를 모아두는 곳을 Store Buffer, LOAD 명령어를 모아두는 곳을 Load Buffer라고 한다.

Store Buffer와 Load Buffer는 서로 통신할 수 있다.  
만약 Load Buffer에서 읽어야 하는 데이터가 Store Buffer에 있다면 Store Buffer에서 가져온다.

![Buffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img.webp)


## 2. Store Buffer
데이터를 메모리(캐시메모리, 물리메모리)에 기록하는 작업은 CPU 입장에서 매우 느린 작업이다.  
그래서 CPU는 메모리에 기록이 완료되는것을 기다리는 대신에 store buffer에 넣어두고 다음 명령을 수행한다.  
store buffer에 있는 데이터는 나중에 규칙에 의해 메모리에 실제로 write 된다.  




![LoadBuffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img-storebuffer.webp)

## 3. Load Buffer

![LoadBuffer](/assets/img/posts/2026-02-05-CPU-Out-of-Order-Execution-img-loadbuffer.webp)


## 2. 인텔 64 아키텍처에서의 Out-of-Order-Execution


https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf
