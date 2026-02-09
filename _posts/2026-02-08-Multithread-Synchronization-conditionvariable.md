---
title: std::condition_variable
date: 2026-02-09 21:00:35 +0900
categories: [멀티스레드, 동기화]
tags: [multithread, synchronization]
description: std::condition_variable
---

## 1. std::condition_variable

condition_variable은 스레드를 대기시켜 놓았다가 원하는 때에 깨우는 기능을 구현할 때 사용한다.  
아래 예제를 통해서 알아보자.  

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

private:
    std::condition_variable m_cv;    // condition variable
    std::mutex m_lock;               // mutex 1개가 반드시 필요하다.

    std::queue< std::shared_ptr<Job> > m_jobQueue;  // 공유자원

public:
    // 작업 얻기(작업을 얻을때까지 스레드가 block됨)
    std::shared_ptr<Job> GetJob()
    {
        // unique_lock 생성(생성하면 일단 lock 걸림)
        std::unique_lock<std::mutex> ulock( m_lock );

        // condition_variable을 기다린다.
        // condition_variable은 내부적으로 ulock.unlock() 하여 lock을 해제한 다음, 다른 스레드가 notify_one을 호출할 때까지 기다린다.
        m_cv.wait( ulock, [ this ]
        {
            // 이 위치로 들어왔으면 다른 스레드가 notify_one을 호출한 것이다.
            // 그리고 이 위치로 들어왔으면 condition_variable이 lock을 획득한 상태이다.
            // 그래서 공유자원을 안전하게 사용할 수 있다.

            // 참고: 다른 스레드가 notify_one을 호출하지 않았는데도 스레드가 깨어나는 때가 있다.
            // 이것을 spurious wakeup 이라고 한다.
            // 그래서 스레드가 깨어났을 때 정말로 임계영역에 진입해도 되는 상태인지를 한번 더 체크해야 한다. (그래서 !m_jobQueue.empty(); 를 체크한다)

            // 여기서 true를 리턴하면 임계영역에 진입하는 것이다.
            // false를 리턴하면 lock을 해제하고 다시 condition_variable을 기다린다.
            return !m_jobQueue.empty();
        } );

        // 임계영역 진입함
        std::shared_ptr<Job> spJob = m_jobQueue.front();
        m_jobQueue.pop();
        return spJob;
    }

    // 작업 입력
    void PublishJob( std::shared_ptr<Job> spJob )
    {
        if ( !spJob )
            return;

        {
            // lock 걸고 공유자원 사용
            std::lock_guard lockGuard( m_lock );
            m_jobQueue.push( spJob );
        }

        // condition_variable을 기다리는 스레드 하나를 깨운다.
        m_cv.notify_one();
    }
};

// 소비자 클래스
class Consumer
{
public:
    Consumer() {}
    ~Consumer() {}

private:
    std::weak_ptr<Producer> m_wpProducer;

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
};

// 생산자, 소비자 생성하기
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
