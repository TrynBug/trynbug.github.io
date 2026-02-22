---
title: Overlapped I/O
date: 2026-02-23 00:50:58 +0900
categories: [OS, I/O]
tags: [OS, I/O]
description: Overlapped I/O
---

## Overlapped I/O
Overlapped IO는 비동기 I/O이다.  
비동기 IO는 OS가 I/O작업을 대신 처리해주는 것이다.  
스레드가 OS에게 I/O 작업을 요청해두고, 작업의 완료를 기다리지 않은 채 다른 작업을 계속해서 수행할 수 있다.  
Overlapped IO라는 이름이 붙은 이유는, 스레드가 I/O 작업이 완료된것을 기다리지 않아도 되기 때문에 OS에게 여러개의 IO를 연속해서 요청할 수 있기 때문이다.  
그래서 사실 Overlapped I/O 라는 이름 대신 Asynchronous I/O(비동기 I/O)라고 부르는것이 더 좋다.  
