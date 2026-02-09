---
title: std::condition_variable
date: 2026-02-08 22:01:24 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: std::condition_variable
---

## 1. std::condition_variable

condition_variable은 스레드를 대기시켜 놓았다가 원하는 때에 깨우는 기능을 구현할 때 사용한다.




## 2. Producer - Consumer 패턴 예제
```cpp
#include <iostream>
#include <thread>
#include <condition_variable>
#include <queue>
#include <chrono>

// 작업
struct Job
{
    int jobType = 0;
};

// 생산자 클래스
class Producer
{
public:
    Producer() {}
    ~Producer() {}

public:
    // 작업 입력
    void PublishJob( std::shared_ptr<Job> spJob )
    {
        if ( !spJob )
            return;

        {
            std::lock_guard lockGuard( m_lock );
            m_jobQueue.push( spJob );
        }

        m_cv.notify_one();
    }

    // 작업 얻기(작업을 얻을때까지 스레드가 block됨)
    std::shared_ptr<Job> GetJob()
    {
        std::unique_lock<std::mutex> ulock( m_lock );

        m_cv.wait( ulock, [ this ]
        {
            return !m_jobQueue.empty();
        } );

        std::shared_ptr<Job> spJob = m_jobQueue.front();
        m_jobQueue.pop();
        return spJob;
    }

private:
    std::condition_variable m_cv;
    std::mutex m_lock;

    std::queue< std::shared_ptr<Job> > m_jobQueue;
};

// 소비자 클래스
class Consumer
{
public:
    Consumer() {}
    ~Consumer() {}

public:
    // 생산자 등록
    void SetProducer( std::shared_ptr<Producer> spProducer ) { m_wpProducer = spProducer; }

    // 생산자로부터 작업을 가져와서 계속해서 처리하기
    void DoWork()
    {
        while ( true )
        {
            std::shared_ptr<Producer> spProducer = m_wpProducer.lock();
            if ( !spProducer )
                return;

            std::shared_ptr<Job> spJob = spProducer->GetJob();
            if ( !spJob )
                continue;

            std::cout << "thread=" << std::this_thread::get_id() << ", \tjob=" << spJob->jobType << std::endl;

            switch ( spJob->jobType )
            {
            case 1:break;
            default: break;
            }
        }
    }

private:

    std::weak_ptr<Producer> m_wpProducer;
};

int main()
{
    std::shared_ptr<Producer> spProducer = std::make_shared<Producer>();

    // 생산자 스레드 생성
    std::thread thProducer = std::thread([ spProducer ]( )
    {
        using namespace std::chrono_literals;
        while ( true )
        {
            // 10ms 마다 작업 생성
            std::this_thread::sleep_for( 10ms );
            
            std::shared_ptr<Job> spJob = std::make_shared<Job>();
            spJob->jobType = rand();

            spProducer->PublishJob( spJob );
        }
    });

    // 소비자 스레드 10개 생성
    std::vector< std::thread > thConsumers;
    for ( int i = 0; i < 10; ++i )
    {
        thConsumers.emplace_back( [ spProducer ]()
        {
            std::shared_ptr<Consumer> spConsumer = std::make_shared<Consumer>();
            spConsumer->SetProducer( spProducer );

            spConsumer->DoWork();
        } );
    }

    thProducer.join();

    return 0;
}
```




## 3. 스레드 루틴을 일정주기로 반복실행 해주는 util 클래스

UserThread 클래스는 등록된 스레드 루틴을 원하는 주기로 반복해서 실행해주는 클래스이다.

```cpp
#include <iostream>
#include <thread>
#include <functional>
#include <future>
#include <condition_variable>

class UserThread
{
public:
    inline static constexpr int THREAD_START_WAIT_SECONDS = 2;    // 스레드 시작 후 시작이 완료되기까지 최대로 기다리는 시간

    UserThread();
    ~UserThread();

public:
    // 스레드 시작하기
    bool StartThread( const std::string& strThreadName, const __int64 nThreadFreqMs, std::function<void()> userThreadProc );

    // 스레드 종료
    void TerminateThread( void );

    // 스레드 루틴 반복주기 설정
    void SetThreadRunFreq( const __int64 nThreadFreqMs );

private:
    // 스레드 루틴
    void threadProc( std::promise<void>&& promiseThreadStart, std::function<void()> userThreadProc );

private:
    std::string              m_strThreadName;        // 스레드 이름
    std::atomic<__int64>     m_nThreadRunFreqMs;     // 스레드 작동주기(단위:밀리초)

    std::thread              m_thread;
    std::atomic<bool>        m_bRunThread;
    std::condition_variable  m_conditionVariable;
    std::mutex               m_lock;
};

UserThread::UserThread()
    : m_nThreadRunFreqMs( 0 ),
    m_bRunThread( false )
{
}

UserThread::~UserThread()
{
}

// 스레드 시작하기
// @strThreadName    : 스레드 이름(로그 출력용)
// @nThreadFreqMs    : 스레드 실행 주기 (ms)
// @userThreadProc   : 사용자 스레드 루틴
bool UserThread::StartThread( const std::string& strThreadName, const __int64 nThreadFreqMs, std::function<void()> userThreadProc )
{
    std::promise<void>    promiseThreadStart;
    std::future<void>    futureThreadStart = promiseThreadStart.get_future();

    const bool bPrevVal = m_bRunThread.exchange( true );
    if ( true == bPrevVal )
    {
        std::cout << strThreadName << " 스레드가 중복으로 시작됨." << std::endl;
        return false;
    }

    m_nThreadRunFreqMs.store( nThreadFreqMs );
    m_strThreadName = strThreadName;

    m_thread = std::thread( &UserThread::threadProc, this, std::move( promiseThreadStart ), userThreadProc );

    std::future_status futureStatus = futureThreadStart.wait_for( std::chrono::seconds( THREAD_START_WAIT_SECONDS ) );
    if ( futureStatus == std::future_status::ready )
    {
        futureThreadStart.get();
        std::cout << m_strThreadName << " 스레드 시작 성공. Freq(ms)=" << m_nThreadRunFreqMs.load() << std::endl;
    }
    else
    {
        std::cout << m_strThreadName << " 스레드 시작에 실패함 !!" << std::endl;

        TerminateThread();
        return false;
    }

    return true;
}

// 스레드 종료
void UserThread::TerminateThread( void )
{
    m_bRunThread.store( false );

    m_conditionVariable.notify_all();

    if ( m_thread.joinable() )
        m_thread.join();

    m_thread = std::thread();
}

void UserThread::SetThreadRunFreq( const __int64 nThreadFreqMs )
{
    m_nThreadRunFreqMs.store( nThreadFreqMs );
}

// 스레드 루틴
// @userThreadProc : 사용자 스레드 루틴
void UserThread::threadProc( std::promise<void>&& promiseThreadStart, std::function<void()> userThreadProc )
{
    promiseThreadStart.set_value();

    std::unique_lock<std::mutex> ulock( m_lock );

    while ( true )
    {
        // 매 timeout마다 스레드가 깨어나지만, 원하는 때에도 스레드를 깨울 수 있게 하기 위해 condition_variable을 사용한다. (멀티스레드 동기화 용도는 아님)
        m_conditionVariable.wait_for( ulock, std::chrono::milliseconds( m_nThreadRunFreqMs.load() ), [ this ]
        {
            return !m_bRunThread.load();
        } );

        if ( false == m_bRunThread.load() )
            break;

        // 작업
        userThreadProc();
    }
}
```
