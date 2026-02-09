---
title: std::atomic::wait
date: 2026-02-09 23:41:21 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: std::atomic::wait
---

## 1. std::atomic::wait

std::atomic::wait은 스레드를 대기시켜 놓았다가 원하는 때에 깨우는 기능을 구현할 때 사용한다.  
변수 1개만을 가지고 구현되기 때문에 매우 가볍다.  

예제를 통해서 사용법을 알아보자.


## 2. 스레드를 한번에 깨우기

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

int main()
{
    std::atomic<bool> bWait{ true };

    // 스레드 10개 생성
    std::vector< std::thread > threads;
    for (int i = 0; i < 10; ++i)
    {
        threads.emplace_back([&bWait, i]()
        {
            // bWait 값이 true인 동안 대기한다.
            while( bWait.load() == true )
            {
                // 값이 true인 동안 대기함
                bWait.wait(true);

                // bWait.wait 함수에서 스레드가 깨어난 다음에 bWait.load() == true 를 한번 더 체크하는 이유는 Spurious Wakeup을 방지하기 위함이다.
                // 다른 스레드가 notify_all을 호출하지 않아도 운영체제에 의해 잘못 깨어날 수 있는데 이것을 Spurious Wakeup 이라고 한다.
            }

            std::cout << "스레드 " << i << "깨어남" << std::endl;
        });
    }

    // 2초간 기다림
    std::this_thread::sleep_for(std::chrono::seconds(2));

    // 값을 false로 바꾸고 모든 스레드를 깨움
    bWait.store(false);
    bWait.notify_all();

    // 종료대기
    for (auto& thread : threads)
    {
        thread.join();
    }

    return 0;
}


```
