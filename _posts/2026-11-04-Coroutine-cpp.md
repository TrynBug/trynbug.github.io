---
title: C++ 코루틴 기본
date: 2026-11-04 23:17 +0900
categories: [코루틴, C++]
tags: [coroutine, c++]
description: C++ 코루틴 기본
---


## 개요
이번 포스트에서는 C++ 코루틴을 실제 코드에서 어떻게 사용해야 하는지를 알아봅니다.  
IOCP worker 스레드에서 DB call을 코루틴으로 처리하는 예제를 살펴봅니다.


## 코루틴 프레임 (coroutine frame)
일반 함수는 Stack 메모리가 필요하지만, 코루틴 함수는 코루틴 프레임이 필요합니다.  
코루틴이 중단(suspend) 되었을 때 함수의 실행 상태를 코루틴 프레임에 저장해야하기 때문입니다.  
코루틴 프레임은 코루틴이 재개(resume) 되었을 때 필요한 정보들을 가지고 있습니다.  
코루틴 함수가 Stack 메모리를 안쓰는 것은 아니고, 코루틴 함수는 처음에는 일반 함수처럼 실행되다가 중간에 suspend 될 때 코루틴 프레임을 생성하여 실행 상태를 저장해둡니다.  
코루틴 프레임에 Stack 메모리가 복사되는것은 아닙니다. 코루틴 프레임은 아래와 같은 정보들을 저장해둡니다.  

```text
Coroutine Frame
├─ 1. 함수 매개변수
│
├─ 2. 지역변수
│  └─ 코루틴이 중단되었다가 재개되는 시점에 필요한 지역변수들을 저장해 둡니다.
│
├─ 3. awaiter 객체
│  └─ awaiter 객체는 코루틴을 중단/재개할때 호출할 함수를 제공합니다.
│
├─ 4. promise 객체
│  └─ promise 객체는 코루틴 함수가 시작/종료될 때 코루틴의 중단 방식을 결정하는 함수를 제공합니다.
│
└─ 5. 컴파일러가 관리하는 실행 상태
   └─ 코루틴 함수가 suspend된 지점. 재개할 때 이 지점에서 시작합니다.
```


## 코루틴 핸들 (std::coroutine_handle<>)
코루틴 핸들은 코루틴 프레임을 가리키는 핸들입니다.  
코루틴 핸들은 코루틴 프레임이 생성될 때 함께 생성되며, 사용자에게 직접 전달됩니다.  
사용자는 코루틴 핸들을 보관하고, 코루틴 핸들을 통해 코루틴 프레임을 제어합니다.  
코루틴 프레임은 사용자에게 노출되지 않습니다.  

```cpp
std::coroutine_handle<> handle;

// 코루틴 핸들로 할 수 있는 대표 작업:
handle.resume();    // 코루틴 재개
handle.destroy();   // 코루틴 프레임 제거
handle.done();      // 코루틴이 완료되었는지 확인
```

## 코루틴 함수
함수 본문에 다음 키워드 중 하나 이상이 있으면 그 함수는 코루틴 함수입니다.  
```cpp
co_await
co_yield
co_return
```

## 실제 코루틴 사용 예제
IOCP worker 스레드에서 DB call을 코루틴으로 처리하는 예제코드

### 예제 코드의 스레드 구조
```text
[main 스레드]
└─ 1초마다 IOCP에 job 등록

[IOCP]
└─ [IOCP worker 스레드 × 2]
   ├─ IOCP 완료 큐에서 작업 꺼내서 처리
   ├─ 코루틴으로 DB Worker에 DB 작업 요청
   └─ DB 작업 완료 후 후속작업 수행

[DB Worker]
└─ [DB worker 스레드 × 4]
   ├─ IOCP worker 스레드가 요청한 DB 작업 수행
   └─ DB 처리 결과를 IOCP에 전달
```

### 예제 코드
예제 코드는 **[#코루틴]** 주석을 번호 순서대로 따라가면서 보면 됩니다.  
코루틴이 실제로 어떻게 진행이 되는지 파악할 수 있습니다.  

- main.cpp

```cpp
#include "DB.h"
#include "IOCP.h"
#include "Contents.h"
#include <mysql.h>
#include <chrono>
#include <exception>
#include <random>


int main()
{
    if (mysql_library_init(0, nullptr, nullptr) != 0)
        return 1;

    int exitCode = 0;
    try
    {
        // MySQL 접속정보 설정
        // 아래쪽의 DB_schema.sql 내용에 DB 스키마가 있다.
        auto config = std::make_shared<DbConfig>();
        config->host = "127.0.0.1";
        config->user = "coroutine_example";
        config->password = "";
        config->port = 3306;

        // DB worker 스레드 4개 시작
        auto db = std::make_shared<DbWorkerPool>(config, 4);

        // IOCP worker 스레드 2개 시작
        auto iocp = Iocp::Instance();
        iocp->Start(2);

        // job 처리 로직이 있는 Contents 객체 생성
        auto contents = std::make_shared<Contents>(db);

        // 1초마다 IOCP에 job을 등록한다.
        while (true)
        {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            
            int r = rand() % static_cast<int>(JobType::MAX);
            const JobType type = static_cast<JobType>(r);

            // IOCP에 job 등록
            iocp->PostJob(type, contents);
        }
    }
    catch (const std::exception& exception)
    {
        Trace(exception.what());
        exitCode = 1;
    }

    mysql_library_end();

    return exitCode;
}
```

- Contents.h

```cpp
#pragma once

#include "DB.h"
#include "IOCP.h"

enum class JobType
{
    PrintMessage,
    LoadGold,
    MAX,
    ResumeCoroutine
};

class Contents;
class DbWorkerPool;

// 컨텐츠 로직 구현하는 클래스
class Contents : public std::enable_shared_from_this<Contents>
{
public:
    explicit Contents(std::shared_ptr<DbWorkerPool> db);
    void Dispatch(JobType type);

private:
    void handlePrintMessage();
    void handleLoadGold();
    static DetachedTask loadGoldAsync(std::shared_ptr<Contents> spContents, std::shared_ptr<IResumeExecutor> executor);

private:
    std::shared_ptr<DbWorkerPool> m_db;
};
```

- Contents.cpp

```cpp
#include "DB.h"
#include "Contents.h"

Contents::Contents(std::shared_ptr<DbWorkerPool> db) : m_db(std::move(db)) {}

// JobType 별로 로직 처리
void Contents::Dispatch(JobType type)
{
    switch (type)
    {
    case JobType::PrintMessage:
        handlePrintMessage();
        break;
    case JobType::LoadGold:
        handleLoadGold();
        break;
    }
}

void Contents::handlePrintMessage()
{
    Trace("PrintMessage job: hello");
}

// IOCP worker 스레드에서 호출되는 job 처리 함수이다. 코루틴을 사용하여 DB 요청을 비동기로 처리한다.
void Contents::handleLoadGold()
{
	std::shared_ptr<Contents> contents = shared_from_this();

    // IResumeExecutor는 코루틴이 재개될 때 어느 스레드에서 재개되는지를 결정하는 컴포넌트이다.
    std::shared_ptr<IResumeExecutor> resumeExecutor = Iocp::Instance()->GetResumeExecutor();

    /* [#코루틴 01 | IOCP worker 스레드] loadGoldAsync 코루틴 함수를 호출한다.
    loadGoldAsync는 코루틴 함수인데, 함수 안에서 co_await을 사용하기 때문이다.
    코루틴 함수를 호출하면 컴파일러가 함수 호출 코드 앞에 아래 준비 절차를 수행하는 코드를 생성한다:
        1. 함수 인자를 평가한다. contents, resumeExecutor 인자를 함수에 전달한다.
        2. 코루틴 프레임을 생성한다. 코루틴 프레임은 코루틴이 시작되고 끝날 때까지 필요한 정보들이 저장되어 있다.
           코루틴 프레임에는 코루틴이 중단된 지점, 재개될 지점, 함수 인자, 현재 함수의 지역변수, awaiter 객체, promise_type 객체 등이 있다.
        3. DetachedTask::promise_type 객체를 생성하여 코루틴 프레임의 promise 멤버에 저장한다.
           DetachedTask 는 loadGoldAsync 함수의 반환 타입인데, DetachedTask는 반드시 promise_type 이라는 이름의 구조체 정의를 가지고 있어야 한다.
           컴파일러는 정확히 promise_type 이름의 구조체를 찾아서 생성한다.
        4. 코루틴 프레임의 promise.get_return_object() 를 호출하여 호출자에게 반환할 DetachedTask 객체를 만든다.
        5. promise.initial_suspend() 를 호출하여 함수 본문을 실행할 지 아니면 코루틴을 중단할지를 결정한다.
           지금 promise.initial_suspend() 함수는 std::suspend_never 를 반환하기 때문에, 코루틴을 중단하지 않고 바로 loadGoldAsync 함수 본문을 실행한다. */
    loadGoldAsync(contents, resumeExecutor);

    // [#코루틴 11 | IOCP worker 스레드] loadGoldAsync 함수에서 co_await 으로 코루틴을 suspend(중단)한 뒤, 여기로 제어가 돌아온다.
	// 그래서 handleLoadGold 함수는 여기서 종료된다. 이 함수가 종료되기 때문에 IOCP worker 스레드는 계속해서 다른 job을 처리할 수 있다.
    // loadGoldAsync 코루틴은 DB 작업이 완료될 때까지 suspend 상태로 남아있다. 코루틴 재개에 필요한 정보는 코루틴 프레임에 저장되어 있다.
}

// 코루틴 함수. DB 작업을 요청한다.
DetachedTask Contents::loadGoldAsync(std::shared_ptr<Contents> spContents, std::shared_ptr<IResumeExecutor> executor)
{
    // [#코루틴 04 | IOCP worker 스레드] 함수 본문이 시작된다. 
    // 이 시점에는 코루틴 프레임이 생성된 상태이다.
    
    Trace("LoadGold job: DB request started");
    auto request = std::make_shared<const DbRequest>(DbRequest{ DbOperation::Read, 1, 0 });

    /* [#코루틴 05 | IOCP worker 스레드] Execute 함수를 co_await 키워드로 호출한다.
    Execute 함수는 awaitable 객체(DbAwaitable 클래스 객체)를 반환하는데, 
    awaitable 객체를 반환하는 함수를 co_await 하면 컴파일러가 awaitable의 await_suspend() 함수를 호출하는 코드를 삽입해준다.
    awaitable은 co_await 할 수 있는 타입을 지칭하는 용어이다.

    co_await 키워드로 함수를 호출하면 컴파일러가 아래 코드를 생성한다:
        1. 함수 인자를 평가하고 함수에 전달한다: request, executor
        2. Execute를 호출하여 DbAwaitable 객체를 생성하고, 코루틴 프레임의 awaiter 멤버에 저장한다.
        3. 코루틴 프레임의 awaiter.await_ready() 함수를 호출한다.
           현재 awaiter.await_ready() 의 반환값은 false 이므로, 4번으로 진행한다.
           만약 반환값이 true 였다면 곧바로 7번으로 진행한다.
        4. 코루틴 프레임에 현재 위치(코루틴 재개 위치)를 기록하고, loadGoldAsync를 suspend(중단) 상태로 만든다.
           그리고 코루틴 프레임에 함수 인자, 지역변수, awaiter를 저장하여 코루틴이 재개될 때까지 수명이 유지되도록 한다.
           이 시점에서 result 변수는 아직 초기화되지 않았다.
        5. awaiter.await_suspend(coroutine_handle) 함수를 호출한다.
           인자로 전달된 coroutine_handle은 현재 코루틴 프레임을 가리키는 핸들이다.
           awaiter.await_suspend(coroutine_handle) 함수의 내용을 확인해보자.
        6. awaiter.await_suspend 함수가 반환되었다.
           그러면 제어는 loadGoldAsync 함수를 호출했던 지점으로 돌아간다.
           즉, Contents::handleLoadGold 함수에서 loadGoldAsync 함수 호출 다음 코드로 돌아간다. */
    auto result = co_await spContents->m_db->Execute(request, executor);
    /*  [#코루틴 20 | IOCP worker 스레드] 누군가 coroutine_handle.resume() 을 호출하여 코루틴이 재개되었다.
        7. awaiter.await_resume() 함수가 호출되었고, 반환값으로 result 변수가 초기화 되었다.
        8. 코루틴 프레임 awaiter에 저장되어있던 DbAwaitable 객체를 파괴한다. */


    // [#코루틴 21 | IOCP worker 스레드] DB 결과를 처리한다.
    if (!result->success)
        Trace("LoadGold job: DB error: " + result->errorMessage);
    else if (result->rows.empty())
        Trace("LoadGold job: player not found");
    else
        Trace("LoadGold job: gold = " + result->rows[0][0]);

    /* [#코루틴 22 | IOCP worker 스레드] 코루틴 함수가 종료된다.
    코루틴 함수가 종료될 때는 컴파일러가 추가한 아래 코드가 실행된다:
        1. promise.return_void() 함수를 호출한다. 
           DetachedTask::promise_type 에는 return_void() 함수가 구현되어 있는데, 이것은 코루틴 함수가 리턴할 값이 없다는 의미이다.
        2. 코루틴 프레임에 저장되어 있던 지역변수들을 정리한다.
        3. promise.final_suspend() 함수를 호출한다.
           현재 final_suspend() 함수 반환타입은 std::suspend_never 이기 때문에 코루틴이 suspend되지 않고 함수가 종료된다.
        4. 코루틴 프레임을 제거한다. */
}
```

- IOCP.h

```cpp
#pragma once

#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <Windows.h>
#include <coroutine>
#include <functional>
#include <memory>
#include <string>
#include <thread>
#include <vector>

// 로그출력
void Trace(const std::string& message);
// 조건체크
void Check(bool condition, const char* message);

// 이 컴포넌트는 코루틴이 중지됐을 때 어떤 스레드에서 재개되어야 하는지 결정할때 사용된다.
class IResumeExecutor
{
public:
    virtual ~IResumeExecutor() = default;
    virtual void Post(std::function<void()> task) = 0;
};

class Iocp;
class Contents;
enum class JobType;

// 코루틴 재개 요청을 IOCP worker 스레드에 전달하는 컴포넌트다.
class IocpResumeExecutor : public IResumeExecutor
{
public:
    explicit IocpResumeExecutor(Iocp& owner) : m_owner(owner) {}
    void Post(std::function<void()> task) override;
private:
    Iocp& m_owner;
};

// IOCP 포트 하나와 그 포트에서 작업을 꺼내 실행하는 worker들을 소유한다.
// 내부 job 객체를 생성하고 completion packet으로 전달한 뒤 처리 완료 시 해제한다.
class Iocp
{
public:
    static std::shared_ptr<Iocp> Instance();

private:
    Iocp() = default;
    Iocp(const Iocp&) = delete;
    Iocp& operator=(const Iocp&) = delete;

public:
    ~Iocp();

public:
    void Start(int workerCount);
    void Stop();

    std::shared_ptr<IResumeExecutor> GetResumeExecutor();

    void PostJob(JobType type, std::shared_ptr<Contents> contents);
    void Post(std::function<void()> task);

private:

    // job
    struct IocpJob
    {
        JobType type;
        std::shared_ptr<Contents> contents;
        std::function<void()> resumeTask;
    };

    
    void postJob(std::unique_ptr<IocpJob> job);

    void run();

private:
    HANDLE m_hIOCP = nullptr;
    std::vector<std::thread> m_workers;
    IocpResumeExecutor m_resumeExecutor { *this };
};
```

- IOCP.cpp

```cpp
#include "DB.h"
#include "IOCP.h"
#include "Contents.h"
#include <iostream>
#include <stdexcept>
#include <syncstream>

void Trace(const std::string& message)
{
	// 로그 출력
    std::osyncstream(std::cout) << "[thread " << GetCurrentThreadId() << "] " << message << '\n';
}

void Check(bool condition, const char* message)
{
    // 조건이 false면 예외를 던진다.
    if (!condition)
        throw std::runtime_error(message);
}

void IocpResumeExecutor::Post(std::function<void()> task)
{
    // [#코루틴 16 | DB worker 스레드] IOCP에 코루틴 재개 작업을 삽입한다.
    m_owner.Post(std::move(task));
}

// singleton
std::shared_ptr<Iocp> Iocp::Instance()
{
    static std::shared_ptr<Iocp> instance(new Iocp());
    return instance;
}

Iocp::~Iocp()
{
    Stop();
}

void Iocp::Start(int workerCount)
{
    Check(m_hIOCP == nullptr, "IOCP already started");

    // completion port 생성
    m_hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, nullptr, 0, workerCount);
    Check(m_hIOCP != nullptr, "CreateIoCompletionPort failed");

    // worker 스레드 생성
    for (int index = 0; index < workerCount; ++index)
        m_workers.emplace_back(&Iocp::run, this);
}

// IOCP에 job 등록
void Iocp::PostJob(JobType type, std::shared_ptr<Contents> contents)
{
    std::unique_ptr<IocpJob> job = std::make_unique<IocpJob>(IocpJob{ type, std::move(contents), {} });
    postJob(std::move(job));
}

std::shared_ptr<IResumeExecutor> Iocp::GetResumeExecutor()
{
    return std::shared_ptr<IResumeExecutor>(Instance(), &m_resumeExecutor); // m_resumeExecutor 의 수명을 Iocp와 공유하도록 함
}

// IOCP에 코루틴 재개 job 등록
void Iocp::Post(std::function<void()> task)
{
    std::unique_ptr<IocpJob> job = std::make_unique<IocpJob>(IocpJob{ JobType::ResumeCoroutine, nullptr, std::move(task) });
    postJob(std::move(job));
}

void Iocp::Stop()
{
    if (m_hIOCP == nullptr)
        return;

    // 완료통지 큐에 key=0인 완료통지를 삽입하여 worker 스레드가 종료되도록 한다.
    for (size_t index = 0; index < m_workers.size(); ++index)
        PostQueuedCompletionStatus(m_hIOCP, 0, 0, nullptr);

    for (auto& worker : m_workers)
        worker.join();

    m_workers.clear();
    CloseHandle(m_hIOCP);
    m_hIOCP = nullptr;
}

void Iocp::postJob(std::unique_ptr<IocpJob> job)
{
    // job을 완료통지 큐에 등록
    BOOL result = PostQueuedCompletionStatus(m_hIOCP, 0, reinterpret_cast<ULONG_PTR>(job.get()), nullptr);

    Check(result != FALSE, "PostQueuedCompletionStatus failed");

    job.release();
}

// IOCP worker 스레드 루틴
void Iocp::run()
{
    while (true)
    {
        DWORD bytes = 0;
        ULONG_PTR key = 0;
        OVERLAPPED* overlapped = nullptr;
        
        // GQCS
		BOOL result = GetQueuedCompletionStatus(m_hIOCP, &bytes, &key, &overlapped, INFINITE);
        Check(result != FALSE, "GetQueuedCompletionStatus failed");

        if (key == 0)
            return;

        // 큐에서 job의 소유권을 회수한다. 이 반복의 처리가 끝나면 job을 자동 해제한다.
        std::unique_ptr<IocpJob> job(reinterpret_cast<IocpJob*>(key));

        if (job->type == JobType::ResumeCoroutine)
        {
            // [#코루틴 17 | IOCP worker 스레드] 
            // 이 job은 [#코루틴 15] 에서 IOCP 완료통지 큐에 등록한 job 이다.
            // 코루틴 재개 함수를 호출한다.
            job->resumeTask();

            // [#코루틴 25 | IOCP worker 스레드] 코루틴이 완료되었고 코루틴 프레임도 정리되었다.
        }
        else
        {
            // 일반 job 처리 루틴
            job->contents->Dispatch(job->type);
        }
    }
}
```

- DB.h

```cpp
#pragma once
#include "DB.h"
#include <condition_variable>
#include <coroutine>
#include <memory>
#include <mutex>
#include <queue>
#include <string>
#include <thread>
#include <vector>

struct MYSQL;
class IResumeExecutor;

// main에서 worker 전용 연결들을 미리 생성할 때 사용하는 공통 설정이다.
struct DbConfig
{
    std::string host;
    std::string user;
    std::string password;
    unsigned int port = 3306;
};

// Read=gold 조회, Write=gold 저장, Fail=오류 결과 반환용 프로시저 호출.
enum class DbOperation { Read, Write, Fail };

// 프로시저 종류와 입력값을 묶은 DB 요청 데이터다. gold는 Write에서만 사용한다.
// 호출부와 DB Job이 shared_ptr<const DbRequest>로 공유하며 처리 중에는 값을 변경하지 않는다.
struct DbRequest
{
    DbOperation operation = DbOperation::Read;
    int playerId = 1;
    int gold = 0;
};

// 프로시저 실행의 성공 여부, 오류 정보, 조회 행을 담는 공통 결과 데이터다.
// DB worker가 내용을 채운 뒤 executor를 통해 전달하면 재개된 코루틴이 읽는다.
struct DbResult
{
    bool success = false;
    unsigned int errorCode = 0;
    std::string errorMessage;
    // 바깥 vector는 행, 안쪽 vector는 각 행의 컬럼 값이다. Write 성공 시에는 비어 있다.
    std::vector<std::vector<std::string>> rows;
};


// 이 타입은 코루틴 함수의 반환 타입으로 사용될 수 있다. 왜냐하면 아래 2가지 조건을 만족하기 때문이다:
// 1. promise_type 이 있다.
// 2. promise_type::get_return_object 함수로 코루틴 함수 반환 타입 객체를 생성할 수 있다.
// 이 타입은 반환값이 없으므로 promise_type::return_void() 를 구현한다.
struct DetachedTask
{
    struct promise_type
    {
        // [#코루틴 02 | IOCP worker 스레드] 호출자에게 반환할 빈 DetachedTask 객체를 만든다.
        DetachedTask get_return_object() noexcept
        {
            return {};
        }

        // [#코루틴 03 | IOCP worker 스레드] 반환값이 suspend_never이므로 코루틴을 중단하지 않고 코루틴함수 본문으로 들어간다.
        std::suspend_never initial_suspend() noexcept
        {
            return {};
        }

        // [#코루틴 23 | IOCP worker 스레드] 코루틴 함수의 반환값은 없다.
        void return_void() noexcept {}

        // [#코루틴 24 | IOCP worker 스레드] 마지막 suspend 없이 promise/함수인자를 정리하고 코루틴 프레임을 해제한다.
        std::suspend_never final_suspend() noexcept 
        { 
            return {}; 
        }
        
        // 코루틴 본문에서 발생한 처리되지 않은 예외가 있으면 이 함수가 호출됨
        void unhandled_exception() noexcept
        {
            std::terminate();
        }

    };
};

class DbWorkerPool;


// 이 클래스 타입은 Awaitable 이다. 이 타입을 반환하는 함수는 co_await 키워드로 호출할 수 있다.
// 이 클래스의 객체는 co_await 키워드로 호출될 때 await_ready, await_suspend, await_resume 세 가지 함수를 통해 코루틴의 중단과 재개를 제어한다.
// Awaitable이 되기 위한 조건은 await_ready, await_suspend, await_resume 3개 함수를 제공하는 것이다.
class DbAwaitable
{
public:
    DbAwaitable(DbWorkerPool& pool, std::shared_ptr<const DbRequest> request, std::shared_ptr<IResumeExecutor> executor);

    // [#코루틴 07 | IOCP worker 스레드] false를 반환하여 코루틴이 suspend 되도록 한다.
    bool await_ready() const noexcept
    {
        return false;
    }

    // [#코루틴 08 | IOCP worker 스레드] 컴파일러가 코루틴을 중단하고 코루틴 프레임의 handle을 넘겨준다.
    void await_suspend(std::coroutine_handle<> handle);

    // [#코루틴 19 | IOCP worker 스레드] DB 처리결과를 반환한다.
    std::shared_ptr<DbResult> await_resume() 
    {
        // m_result 에는 DB worker 스레드가 채워준 DB처리 결과값이 들어있다.
        return m_result; 
    }

private:
    DbWorkerPool& m_pool;
    std::shared_ptr<const DbRequest> m_request;
    std::shared_ptr<IResumeExecutor> m_executor;
    std::shared_ptr<DbResult> m_result;
};

// 공용 요청 큐와 DB worker들을 관리한다. 각 worker는 전용 MySQL 연결로 프로시저를 동기 호출한다.
// 완료된 요청은 요청자가 지정한 executor에 재개 작업을 등록한다.
class DbWorkerPool
{
public:
    // main에서 연결을 모두 연 뒤 worker를 시작한다. 연결 실패 시 예외로 초기화를 중단한다.
    DbWorkerPool(std::shared_ptr<const DbConfig> config, int workerCount);
    ~DbWorkerPool();
    // 호출부가 co_await할 대기 객체를 만든다. 요청 등록은 await_suspend에서 이루어진다.
    DbAwaitable Execute(std::shared_ptr<const DbRequest> request, std::shared_ptr<IResumeExecutor> executor);
    // 하나의 Job으로 묶어 큐에 추가하고 대기 중인 DB worker를 깨운다.
    void Submit(std::shared_ptr<const DbRequest> request, std::shared_ptr<DbResult> result,
        std::shared_ptr<IResumeExecutor> executor, std::coroutine_handle<> handle);
    void Stop();
private:
    void run(MYSQL* connection);
    struct Job
    {
        // 입력, 출력, 재개 위치, 재개할 코루틴을 한 작업으로 묶어 큐에 넘긴다.
        std::shared_ptr<const DbRequest> request;
        std::shared_ptr<DbResult> result;
        std::shared_ptr<IResumeExecutor> executor;
        // handle은 프레임의 소유권이나 참조 카운트를 제공하지 않는 비소유 핸들이다.
        // DB 완료 시 한 번 재개한다. 그때까지 대상 프레임은 유효해야 한다.
        std::coroutine_handle<> handle;
    };
    std::mutex m_mutex;
    std::condition_variable m_ready;
    std::queue<Job> m_jobs;
    bool m_stopped = false;
    // index번째 연결은 index번째 worker 전용이다. worker join 이후 main에서 해제한다.
    std::vector<std::shared_ptr<MYSQL>> m_connections;
    std::vector<std::thread> m_workers;
};
```

- DB.cpp

```cpp
#include "DB.h"
#include "IOCP.h"
#include <mysql.h>
#include <stdexcept>
#include <utility>

DbAwaitable::DbAwaitable(DbWorkerPool& pool, std::shared_ptr<const DbRequest> request,
    std::shared_ptr<IResumeExecutor> executor)
    : m_pool(pool), m_request(std::move(request)), m_executor(std::move(executor)), m_result(std::make_shared<DbResult>())
{
    Check(m_request && m_executor, "DB request and resume executor are required");
}

void DbAwaitable::await_suspend(std::coroutine_handle<> handle)
{
    // [#코루틴 09 | IOCP worker] 현재 코루틴은 이미 suspend된 상태이다. 
    // DB job을 등록한다.
    m_pool.Submit(
        m_request,  // 프로시저 입력 데이터.
        m_result,   // DB worker가 채울 결과 버퍼.
        m_executor, // 재개 작업을 실행할 IOCP executor.
        handle);    // 중단된 코루틴 프레임의 비소유 핸들.

    // 주의: job 등록 직후 다른 IOCP worker에서 재개/종료될 수 있으므로 이후 this에 접근하지 않는다.
}

// DB worker 스레드 시작하기
DbWorkerPool::DbWorkerPool(std::shared_ptr<const DbConfig> config, int workerCount)
{
    // DB connection 만들기
    for (int index = 0; index < workerCount; ++index)
    {
        std::shared_ptr<MYSQL> connection(mysql_init(nullptr), mysql_close);
        Check(connection != nullptr, "mysql_init failed");

        unsigned int timeout = 5;
        mysql_options(connection.get(), MYSQL_OPT_CONNECT_TIMEOUT, &timeout);
        mysql_options(connection.get(), MYSQL_OPT_READ_TIMEOUT, &timeout);
        mysql_options(connection.get(), MYSQL_OPT_WRITE_TIMEOUT, &timeout);

        if (mysql_real_connect(connection.get(), config->host.c_str(), config->user.c_str(),
            config->password.c_str(), "CoroutineExample", config->port, nullptr, CLIENT_MULTI_RESULTS) == nullptr)
        {
            throw std::runtime_error("DB connection " + std::to_string(index) + " initialization failed: " + mysql_error(connection.get()));
        }

        m_connections.push_back(std::move(connection));
    }

    // DB worker 스레드 시작하기
    try
    {
        for (int index = 0; index < workerCount; ++index)
            m_workers.emplace_back(&DbWorkerPool::run, this, m_connections[index].get());
    }
    catch (...)
    {
        Stop();
        throw;
    }
}

DbWorkerPool::~DbWorkerPool()
{
    Stop();
}

DbAwaitable DbWorkerPool::Execute(std::shared_ptr<const DbRequest> request, std::shared_ptr<IResumeExecutor> executor)
{
    // [#코루틴 06 | IOCP worker] awaitable 객체(DbAwaitable 객체)를 생성한다.
    // DbAwaitable 에는 request, executor를 저장해둔다. 그리고 DbAwaitable 에는 DB 결과를 담을 멤버변수(m_result)도 준비되어 있다.
    return DbAwaitable(*this, std::move(request), std::move(executor));
}

void DbWorkerPool::Submit(std::shared_ptr<const DbRequest> request, std::shared_ptr<DbResult> result,
    std::shared_ptr<IResumeExecutor> executor, std::coroutine_handle<> handle)
{
    // [#코루틴 10 | IOCP worker 스레드] DB 작업 요청을 큐에 넣고 DB worker 스레드를 깨운다.
    {
        std::lock_guard lock(m_mutex);
        Check(!m_stopped, "DB request after shutdown");

        // DB job을 큐에 넣는다.
        m_jobs.push(Job{
            std::move(request),   // DB 요청 데이터
            std::move(result),    // DB 처리결과를 저장할 버퍼
            std::move(executor),  // 코루틴 재개 작업을 등록할 큐
            handle });            // 코루틴 프레임의 핸들
    }

    // DO worker 스레드 깨우기
    m_ready.notify_one();
}

void DbWorkerPool::Stop()
{
    // 신규 Submit을 막고 남은 큐를 처리한 뒤 worker를 join한다.
    // worker 종료는 resume 작업의 등록 완료를 뜻할 뿐, executor의 후속 처리 완료까지 보장하지 않는다.
    // 안전하게 종료하려면 호출자가 새 job 등록을 멈추고 진행 중인 코루틴 완료를 먼저 보장해야 한다.
    {
        std::lock_guard lock(m_mutex);
        m_stopped = true;
    }
    m_ready.notify_all();
    for (auto& worker : m_workers)
    {
        if (worker.joinable())
            worker.join();
    }
    // 모든 worker가 연결 사용을 끝냈으므로 main에서 전용 연결들을 닫는다.
    m_connections.clear();
}


// DB worker 스레드 루틴
void DbWorkerPool::run(MYSQL* connection)
{
    Check(mysql_thread_init() == 0, "mysql_thread_init failed");

    // loop
    while (true)
    {
        Job job;
        {
            std::unique_lock lock(m_mutex);

            // wait
            m_ready.wait(lock, [this] { return m_stopped || !m_jobs.empty(); });

            if (m_jobs.empty())
                break;

            // [#코루틴 12 | DB worker 스레드] 큐에서 DB Job을 꺼낸다.
            job = std::move(m_jobs.front());
            m_jobs.pop();
        }

        Trace("DB worker: synchronous CALL");

        // job 종류에 맞는 프로시저 호출문을 만든다.
        std::string sql;
        switch (job.request->operation)
        {
        case DbOperation::Read:
            sql = "CALL ReadGold(" + std::to_string(job.request->playerId) + ")";
            break;
        case DbOperation::Write:
            sql = "CALL WriteGold(" + std::to_string(job.request->playerId) + "," + std::to_string(job.request->gold) + ")";
            break;
        case DbOperation::Fail:
            sql = "CALL FailExample()";
            break;
        }

        // [#코루틴 13 | DB worker 스레드] DB 프로시저를 동기로 실행하고 결과를 job.result에 채운다.
		// 여기서 job.result 는 [#코루틴 09] 에서 전달된 std::shared_ptr<DbResult> 이다.
        int status = mysql_query(connection, sql.c_str());
        while (status == 0)
        {
            MYSQL_RES* rows = mysql_store_result(connection);
            if (rows != nullptr)
            {
                while (MYSQL_ROW row = mysql_fetch_row(rows))
                {
                    std::vector<std::string> values;

                    // MySQL 결과 버퍼가 해제되어도 결과를 읽을 수 있도록 std::string으로 복사한다.
                    const auto lengths = mysql_fetch_lengths(rows);
                    for (unsigned int column = 0; column < mysql_num_fields(rows); ++column)
                    {
                        if (row[column])
                        {
                            values.emplace_back(std::string(row[column], lengths[column]));
                        }
                        else
                        {   
                            // SQL NULL은 문자열 "NULL"로 저장
                            values.emplace_back("NULL");
                        }
                        
                    }

                    // job.result 에 DB 결과를 채운다.
                    job.result->rows.push_back(std::move(values));
                }

                mysql_free_result(rows);
            }
            // UPDATE, INSERT만 있는 프로시저 등은 결과셋이 없다. 결과셋이 있어야 하는데 없을 때만 오류다.
            else if (mysql_field_count(connection) != 0)
            {
                status = 1;
                break;
            }

            // mysql_next_result: 0=다음 결과셋 있음, -1=모두 소비 완료, 양수값=오류코드
            status = mysql_next_result(connection);
        }

        if (status == -1)
        {
            job.result->success = true;
        }
        else
        {
            job.result->success = false;
            job.result->errorCode = mysql_errno(connection);
            job.result->errorMessage = mysql_error(connection);
            job.result->rows.clear();
        }

        // [#코루틴 14 | DB worker 스레드] 코루틴 재개 작업을 만든다.
        // 여기서 job.handle은 [#코루틴 09] 에서 전달된 std::coroutine_handle 이다.
        // handle.resume() 을 호출하면 코루틴이 재개된다.
		// 주의: handle.resume() 호출 이후에는 job.handle을 사용하지 않는다. 코루틴이 재개되면 프레임이 파괴될 수 있기 때문이다.
        auto resume = [handle = job.handle] 
            { 
                // [#코루틴 18 | IOCP worker 스레드] IOCP worker 스레드에서 이 람다함수를 호출하였다.
                // handle.resume()은 코루틴을 재개한다. 
                // 코루틴을 재개하면 코루틴 프레임에 저장되어 있던 중단 지점으로 제어가 이동한다. 중단 지점에서부터 실행을 계속해나간다.
                // 중단 지점으로 제어가 이동하면 그곳에 코루틴 프레임의 awaitable.await_resume 함수 호출 코드를 컴파일러가 삽입해놓았다.
                // 그래서 awaitable.await_resume 함수가 호출된다.
                handle.resume(); 
            };

        // [#코루틴 15 | DB worker 스레드] 코루틴 재개 작업을 job.executor를 통해 IOCP 완료통지 큐에 등록한다. 
        // 여기서 job.executor 는 [#코루틴 09] 에서 전달된 std::shared_ptr<IocpResumeExecutor> 이다.
        // IocpResumeExecutor 는 코루틴이 어느 스레드에서 재개되어야 하는지를 결정하는 역할을 한다(여기서는 IOCP worker 스레드).
        // C++ 코루틴은 코루틴이 suspend된 스레드에서 재개되는것을 보장해주지 않는다. 사용자가 직접 그렇게 만들어야 한다.
        job.executor->Post(std::move(resume));
    }


    mysql_thread_end();
}
```

- DB_schema.sql

```sql
-- 코루틴 예제 DB 스키마
CREATE DATABASE IF NOT EXISTS CoroutineExample;
USE CoroutineExample;

CREATE TABLE IF NOT EXISTS Players (
    player_id INT PRIMARY KEY,
    gold INT NOT NULL
);
INSERT IGNORE INTO Players VALUES (1, 0), (2, 0), (3, 0);

DROP PROCEDURE IF EXISTS ReadGold;
DROP PROCEDURE IF EXISTS WriteGold;
DROP PROCEDURE IF EXISTS FailExample;
DELIMITER //
CREATE PROCEDURE ReadGold(IN playerId INT)
BEGIN
    SELECT gold FROM Players WHERE player_id = playerId;
END//
CREATE PROCEDURE WriteGold(IN playerId INT, IN newGold INT)
BEGIN
    UPDATE Players SET gold = newGold WHERE player_id = playerId;
END//
CREATE PROCEDURE FailExample()
BEGIN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Intentional coroutine example error';
END//
DELIMITER ;
```
