---
title: MMO 게임서버 / 클라이언트 개발 프로젝트
date: 2026-07-29 11:12:49 +0900
categories: [프로젝트, 게임서버]
tags: [project, gameserver]
description: 개인 MMORPG 게임서버 / 클라이언트 개발 프로젝트
---

## 프로젝트 개요

Unity 클라이언트와 C++ MMORPG 서버를 함께 개발하는 프로젝트 입니다.  

| 구분       | 내용                                           |
| ---------- | ---------------------------------------------- |
| 서버       | C++ 20 · Windows · IOCP · MySQL 8.4 · Protobuf |
| 클라이언트 | Unity 6.4 · C# 11.0                            |
| 컨텐츠     | Recast/Detour · Lua 5.4 · sol2                 |
| 모니터링   | Prometheus · Grafana                           |

- 개발기간: 2026.04.24 ~ 2026.07.30
- 소스코드 : https://github.com/TrynBug/MMOGameServerProject

### 프로젝트 개발의 목표
- 적은 컴퓨팅 자원으로도 안정적이고 효율적으로 돌아가는 서버를 설계하고 검증합니다.

### AI 활용
- 이 프로젝트는 AI를 적극적으로 활용하여 개발되었습니다.  
- AI에게 정확한 개발 지시를 하기 위해 여러 설계 문서들을 사용하였습니다.  
설계 문서는 github repository의 Document 폴더 참조.

## 게임플레이 소개

- 필드 위에 다수의 플레이어가 돌아다니면서 사냥하는 게임입니다.
- 플레이어의 위치, 행동은 서버를 통해 실시간으로 공유됩니다.
- 게임플레이 영상

## 서버 전체 구조
![drawio](/assets/img/posts/2026-07-29/GameServer-서버전체구조.drawio.svg)
- **클라이언트**  
  로그인 서버에서 인증받은 후 게이트웨이 서버에 접속합니다. 모든 게임 패킷은 게이트웨이 서버를 통해 송수신합니다.
- **레지스트리 서버**  
  실행 중인 서버의 주소와 상태를 관리합니다. 서버 등록, Heartbeat 및 서버 목록 공유를 담당합니다.
- **로그인 서버**  
  AccountDB에서 계정 정보를 조회하고 로그인 요청을 검증합니다. 접속할 Gateway를 선택하고 인증 토큰을 발급합니다.
- **게이트웨이 서버 (N개)**  
  클라이언트 연결을 관리합니다. 클라이언트 패킷을 배정된 게임서버로 전달하고, 게임서버의 패킷을 클라이언트로 중계합니다.
- **게임 서버 (M개)**  
  Stage, 이동, 전투, 스킬과 같은 핵심 게임 로직을 처리합니다. 유저 상태를 관리하고 게임데이터를 DB에 저장합니다.
- **커뮤니케이션 서버**  
  여러 게임서버 사이의 전체 채팅과 귓속말을 중계합니다.
- **계정 DB**  
  계정 정보와 해당 계정이 사용하는 게임DB 샤드 번호를 저장합니다. 로그인과 게임서버 입장 시 조회됩니다.
- **게임 DB (샤딩)**  
  캐릭터, 아이템, 재화 등 게임플레이 데이터를 분산 저장합니다. DB는 N개로 샤딩되어 있으며, 계정별로 데이터가 몇번째 샤드에 저장되는지가 미리 정해져 있습니다.
- **Prometheus**  
  각 서버의 모니터링 metric을 주기적으로 수집하고 시계열 데이터로 저장합니다. 수집한 지표를 바탕으로 Alert Rule도 평가합니다.
- **Grafana**  
  Prometheus에 저장된 metric을 Dashboard로 시각화합니다. 서버 상태를 한 화면에서 확인할 수 있습니다.

### 레지스트리 서버의 역할
> 레지스트리 서버는 서버들의 목록과 상태를 관리하는 서버입니다.  
> 다른 모든 서버들은 레지스트리 서버로부터 서버 목록을 얻어와 사용합니다.  
> 서버가 가동될 때 반드시 레지스트리 서버에 먼저 등록을 해야 합니다. 등록하지 않으면 서버가 가동되지 않습니다.  

![drawio](/assets/img/posts/2026-07-29/GameServer-레지스트리 서버.drawio.svg){: width="800"}

## 클라이언트의 로그인 절차
![drawio](/assets/img/posts/2026-07-29/GameServer-로그인 절차.drawio.svg)
- 클라이언트는 최초에 로그인서버에 접속하여 로그인을 수행합니다.  
- 그런다음 로그인서버의 연결을 끊고 게이트웨이서버에 접속하여 게임을 플레이 합니다.  
- 게이트웨이서버는 클라이언트 패킷을 게임서버로 전달하고, 게임서버의 패킷을 클라이언트로 중계합니다.
- 실질적인 게임플레이가 이루어지는 곳은 게임서버 입니다.

## 게임서버의 구조
### 게임서버의 네트워크 스레드 구조
![drawio](/assets/img/posts/2026-07-29/GameServer-네트워크 스레드.drawio.svg)
- 게이트웨이 서버로부터 전달된 클라이언트 패킷은 해당 유저객체의 패킷 큐에 저장됩니다. 패킷 큐의 패킷은 나중에 Stage가 Update될 때 Stage 내에서 처리됩니다.
- 다른 서버(게이트웨이서버 포함)로부터 전달된 서버 패킷은 IOCP Worker 스레드에서 직접 처리됩니다.

### 게임서버의 컨텐츠 스레드 구조
> 설계 목표: 각각의 Stage가 반드시 1개 스레드에서 처리되도록 하여 Stage 내부에서 Lock을 제거합니다.  

![drawio](/assets/img/posts/2026-07-29/GameServer-컨텐츠 스레드.drawio.svg){: width="700"}
- 컨텐츠 스레드는 Stage를 소유하고 업데이트하는 스레드 입니다.
- Stage는 마을, 필드, 던전 등의 유저가 입장 가능하고 게임플레이가 가능한 하나의 컨텐츠를 의미합니다.
- 게임서버는 Stage를 컨텐츠 스레드에 분산 배치하며, 하나의 Stage는 반드시 하나의 컨텐츠 스레드에만 배정됩니다. 
- Stage는 반드시 배정된 컨텐츠 스레드 내에서만 업데이트 되며, 다른 스레드에서 접근할 수 없습니다.
- 컨텐츠 스레드는 50ms 마다(초당 20tick) Stage를 업데이트 합니다.

### 게임서버의 Stage 인스턴스 구조
![drawio](/assets/img/posts/2026-07-29/GameServer-Stage 구조.drawio.svg)
- **Stage**  
  Stage는 하나의 독립적인 게임 공간입니다. Stage는 반드시 1개의 컨텐츠 스레드에만 배정되며, 해당 컨텐츠 스레드에서만 Update할 수 있습니다.  
  Stage는 Update될 때 아래 모든 요소들을 Update 합니다.
- **시스템 메시지 큐**  
  `유저가 Stage에 입장`, `유저가 Stage에서 나감` 등의 서버 내부 작업이 입력되는 메시지 큐 입니다.
- **코루틴 Resume Task 큐**  
  Stage 내에서 코루틴을 사용하여 DB 작업을 요청했을 때, 코루틴이 resume 되었을 때 수행할 로직이 이 큐에 저장됩니다.
- **유저 Map**  
  Stage가 소유하는 유저 입니다. 유저는 게임내에서 반드시 1개 Stage에만 배정됩니다. Stage가 Update될 때 유저 객체의 패킷 큐에서 패킷을 꺼내 처리합니다.
- **스테이지 오브젝트**  
  Stage가 소유하는 게임 오브젝트 들입니다. 캐릭터, 몬스터, 프랍, 이벤트영역 등이 있습니다.
- **Sector**  
  Stage내의 이동가능한 공간을 Sector Grid로 나누어 관리합니다.  
  스테이지 오브젝트가 필드 위에서 이동할 때는 NavMesh 데이터와 Detour 라이브러리를 통해 길을 찾습니다. NavMesh는 Unity Editor를 통해 export 되며 서버와 클라이언트가 서로 공유하는 데이터 입니다.  
- **게임 로직**  
  게임 로직은 Lua VM을 통해 진행됩니다. Stage가 생성될 때 자신만의 Lua VM을 하나 생성합니다. Stage는 Lua VM에 API 함수들을 등록하여 Lua Script에서 함수를 사용할 수 있게 합니다.  
  기획자가 게임로직을 설명하는 Lua Script를 작성하면, Stage는 매 tick 마다 Lua Script를 Lua VM을 통해 실행하여 게임 로직을 실행합니다.  
  Lua Script에서는 `특정 위치에 몬스터 스폰`, `이벤트 영역에 플레이어 입장 시 메시지 전송`, `프랍 상호작용 시 몬스터 스폰`, `매 Tick마다 몬스터 스폰` 등의 게임 로직을 작성합니다.  
  Stage Layout은 Stage 위의 프랍, 몬스터 스폰 마커 등의 위치가 기록된 데이터입니다. Unity Editor에서 Stage 위에 프랍, 몬스터 스폰 마커 등을 배치한다음 Stage Layout 데이터로 export 하면 서버가 가져다 사용합니다.  


### 게임서버의 코루틴 DB처리 절차
> 설계 목표: 코루틴을 사용하여서 DB 처리 때문에 코드의 흐름이 끊기는 일 없이 하나의 함수 내에서 계속 이어지도록 합니다.  
> 코루틴이 suspend 되었다가 resume 될 때 반드시 처음 호출되었던 스레드에서 resume 되도록 합니다.  

![drawio](/assets/img/posts/2026-07-29/GameServer-DB 처리절차.drawio.svg)
게임서버에서는 DB 처리를 요청할 때 코루틴을 사용합니다.  
코루틴의 시작점은 DetachedCoTask 함수 입니다. DetachedCoTask 함수는 내부적으로 **co_await** 키워드로 DB요청함수(코루틴 함수)를 실행합니다.  
DetachedCoTask 함수는 fire-and-forget 역할을 하기 때문에 DetachedCoTask 함수의 호출자는 코루틴이 완료되기를 기다리지 않고 즉시 다음 코드를 수행할 수 있습니다.  

**co_await** 키워드로 호출한 DB요청함수는 Awaitable 객체를 생성하여 반환합니다.  
Awaitable 객체는 컴파일러가 요구하는 코루틴 규약을 구현한 객체입니다.  
컴파일러는 **co_await** 키워드를 만나면 현재 함수의 컨텍스트를 캡처하여 coroutine_handle을 생성하고, 앞에서 생성된 Awaitable 객체의 await_suspend(coroutine_handle) 함수를 호출합니다.  
await_suspend 함수 내부에서는 아래 작업을 합니다:  
- DB 처리가 완료된 뒤 호출할 콜백 람다함수 생성: 이 람다함수는 coroutine_handle, 호출자 Stage 정보를 캡처하여 가지고 있습니다.  
- 그런다음 DB Task Queue에 처리할 DB 데이터와 콜백 람다함수를 삽입합니다.  

이 때 코루틴이 완료되기 전까지 Stage 객체와 캐릭터 객체는 파괴되면 안되기 때문에 AsyncPin으로 객체의 수명을 유지시킵니다.  
await_suspend 함수가 종료되면 DetachedCoTask 함수가 suspend 되고, 제어가 DetachedCoTask 함수 호출자에게로 반환되어 함수 호출자가 코드를 계속 수행할 수 있습니다.  

DB Worker 스레드가 DB Task Queue에 있는 DB Task를 꺼내어 DB 작업을 처리합니다.  
처리가 완료되면 Task에 부가적으로 저장되어 있던 콜백 람다함수를 호출합니다. 이 때 람다함수에 DB 처리결과를 인자로 전달합니다.  
콜백 람다함수는 DB 처리결과를 Awaitable 객체에 저장하고, coroutine_handle을 Stage의 코루틴 Resume Queue에 삽입합니다.  

Stage가 Update될 때 코루틴 Resume Queue에서 coroutine_handle을 꺼내어 coroutine_handle.resume() 함수를 호출합니다.  
그러면 **co_await** 키워드로 DB요청함수를 호출한 시점으로 제어가 이동하고, DetachedCoTask 함수가 suspend 되었던 지점에서부터 재개됩니다.  
이 때 Awaitable 객체에 저장되어 있던 DB 처리결과를 전달받아서 코드에서 DB 처리결과를 확인하여 후속조치를 합니다.  


### Stage가 Lua VM을 통해 게임로직 돌리는 방식
> 설계 목표: 기획자가 게임 로직을 Lua Script로 편리하게 작성할 수 있도록 합니다.

![drawio](/assets/img/posts/2026-07-29/GameServer-Lua.drawio.svg){: width="600"}
- 각 Stage 인스턴스는 독립적인 `Lua VM`을 하나씩 소유합니다. Stage는 Lua VM에 API 함수를 제공하고, Lua Script는 Stage가 요구하는 콜백 함수들을 구현합니다.
- 하나의 Stage에서 여러개의 Lua Script를 돌릴 수 있도록 Lua VM에 Script별 `_ENV`를 생성하여 전역 변수와 진행 상태가 서로 충돌하지 않도록 격리합니다.
- Stage 코드 내에서 Lua VM에 Callback 함수를 호출할 때는 `protected_function`으로 실행하여 Script 오류가 서버 전체로 전파되지 않도록 격리합니다. 
- Lua Script 명령어 실행 횟수 제한을 적용하여 무한 반복과 과도한 Script 실행으로부터 Contents Thread를 보호합니다.

<details markdown="1">
<summary>Lua Script 샘플 보기</summary>

```lua
--[[
    Stage Script 샘플

    스크립트 흐름
      1. 플레이어가 시작 영역에 진입한다.
      2. 일반 몬스터 스포너를 활성화한다.
      3. 일반 몬스터를 모두 처치하면 보스 스포너를 활성화한다.
      4. 보스를 처치하면 보상 상자를 생성한다.
      5. 플레이어가 보상 상자와 상호작용하면 일정 시간 뒤 상자를 제거한다.
]]

-- ─────────────────────────────────────────────────────────────
-- Stage Layout Key: Unity에서 배치한 Stage Layout 오브젝트의 식별자
-- ─────────────────────────────────────────────────────────────
local START_EVENT_AREA_KEY   = 1001
local NORMAL_SPAWNER_KEY     = 2001
local TIMER_SPAWNER_KEY      = 2002
local BOSS_SPAWNER_KEY       = 2002
local REWARD_SPAWN_POINT_KEY = 3001

-- GameData Key: 서버 게임 데이터에 정의된 몬스터/Prop 식별자
local BOSS_MONSTER_KEY       = 9000
local REWARD_PROP_DATA_KEY   = 5001

-- 동적으로 생성한 보상 상자를 콜백에서 식별하기 위한 배치 Key
local REWARD_PLACEMENT_KEY   = 4001

-- Stage 인스턴스별 진행 상태
local dungeonStarted = false
local bossDefeated = false
local enteredPlayers = {}
local playerCount = 0


-- 던전 진행 순서를 코루틴으로 직관적으로 표현한다.
-- Wait 계열 함수가 yield하면 StageScript::Update가 조건 충족 후 다시 실행한다.
local function RunDungeonSequence()
    Notice("몬스터를 모두 처치하세요!", 3000)

    -- 이 샘플에서는 1회 웨이브로 설정된 Manual Spawner를 사용한다.
    ActivateSpawner(NORMAL_SPAWNER_KEY)
    WaitForSpawnerClear(NORMAL_SPAWNER_KEY)

    Notice("잠시 후 보스가 등장합니다.", 3000)
    Wait(2000)

    ActivateSpawner(BOSS_SPAWNER_KEY)
    Notice("보스가 등장했습니다!", 3000)

    -- 해당 monsterKey가 사망할 때까지 코루틴을 중단한다.
    WaitForMonsterDead(BOSS_MONSTER_KEY)

    bossDefeated = true
    DeactivateSpawner(BOSS_SPAWNER_KEY)
    Notice("보스를 처치했습니다. 보상 상자가 생성됩니다!", 5000)
	
	-- 보상 상자를 생성한다.
    local point = GetSpawnPoint(REWARD_SPAWN_POINT_KEY)
    local objectId = SpawnProp(
        REWARD_PROP_DATA_KEY,
        point.x, point.y, point.z,
        point.yaw,
        REWARD_PLACEMENT_KEY,
        0                       -- initialState
    )
	
    Log("보상 상자 생성 완료. objectId=" .. objectId)
end


-- 콜백함수: Stage가 시작될 때 한 번 호출되는 초기화 진입점이다.
function OnStageStart()

    -- 5초마다 실행되는 타이머를 등록한다.
    timerId = RegisterTimer(5000, function()
		ActivateSpawner(TIMER_SPAWNER_KEY)
    end)

    -- 보스 몬스터가 사망했을 때 OnMonsterDead 콜백이 호출되도록 미리 등록해둔다.
    WatchMonsterDeath(BOSS_MONSTER_KEY)
end

-- 콜백함수: 캐릭터가 Stage에 스폰된 뒤 호출된다.
function OnPlayerEnter(objectId)
    if enteredPlayers[objectId] then
        return
    end

    enteredPlayers[objectId] = true
    playerCount = playerCount + 1

    Log("플레이어 입장. objectId=" .. objectId .. ", 인원=" .. playerCount)
    Notice("시작 지점으로 이동해 던전을 시작하세요.", 3000)
end

-- 콜백함수: 플레이어가 Stage에서 퇴장할 때 호출된다.
function OnPlayerLeave(objectId)
    if enteredPlayers[objectId] then
        enteredPlayers[objectId] = nil
        playerCount = playerCount - 1
    end

    Log("플레이어 퇴장. objectId=" .. objectId .. ", 인원=" .. playerCount)
end


-- 콜백함수: 플레이어가 EventArea 에 진입했을 때 호출된다.
function OnEnterEventArea(eventKey, objectId, x, y, z)
    if eventKey ~= START_EVENT_AREA_KEY or dungeonStarted then
        return
    end

    dungeonStarted = true
    Log("던전 시작 영역 진입. objectId=" .. objectId)

    -- 첫 yield까지 즉시 실행되고 이후에는 Stage Update에서 재개된다.
    StartSequence(RunDungeonSequence)
end


-- 콜백함수: WatchMonsterDeath로 등록한 보스가 사망했을 때만 호출된다.
function OnMonsterDead(objectId, monsterKey, killerObjectId)
    if monsterKey ~= BOSS_MONSTER_KEY then
        return
    end

    Log("보스 사망. objectId=" .. objectId .. ", killerObjectId=" .. killerObjectId)
end


-- 콜백함수: 플레이어가 프랍을 인터랙션 했을 때 호출된다.
function OnObjectInteract(propObjectId, placementKey, actorObjectId, newState)
    if placementKey ~= REWARD_PLACEMENT_KEY or not bossDefeated then
        return
    end

    Log("보상 상자 상호작용. actorObjectId=" .. actorObjectId .. ", state=" .. newState)
    Notice("보상 상자를 열었습니다!", 3000)

    -- 2초 뒤 상자를 제거한다.
    SetTimeout(2000, function()
        DespawnProp(propObjectId)
    end)
end
```

</details>


## 더미 테스트
> 서버 안정성 확인을 위해 더미 테스트를 수행합니다.

### AWS 인스턴스 구성
![drawio](/assets/img/posts/2026-07-29/GameServer-더미테스트.drawio.svg)
![drawio](/assets/img/posts/2026-07-29/AWS_EC2.png)
_AWS EC2 스크린샷_
![drawio](/assets/img/posts/2026-07-29/AWS_VPC.png)
_AWS VPC 스크린샷_

- AWS 에서 2개의 VPC(서버용, 더미클라이언트용)를 생성하여 네트워크가 서로 분리되도록 구성하였습니다.
- machine 들의 spec은 모두 **m7i-flex.large (2 vCPU, 8 GiB 메모리)** 사용, Windows Server 2022 사용하였습니다.
- 1500 명의 더미 클라이언트로 6시간 동안 테스트하였습니다.

### 더미테스트 결과
- 6시간 동안의 테스트 결과 서버의 CPU 사용률, 메모리 사용률이 안정적으로 유지되었습니다.
- 더미클라이언트에서 특별한 오류가 감지되지 않았습니다.


![drawio](/assets/img/posts/2026-07-29/Grafana.PNG)
_6시간 동안의 Grafana 대시보드 스크린샷_
- **User Counts**  
  게임서버 3대에 500명씩, 게이트웨이서버 2대에 750명씩 입장해 있습니다.
- **Network Throughput**  
  네트워크 사용량은 게임서버 3대 합쳐서 초당 recv 167 kB/s, send 2.18 MB/s 입니다. 게이트웨이서버 2대 합쳐서 초당 recv 2.31 MB/s, send 2.38 MB/s 입니다.
- **Network Backlog**  
  서버의 네트워크 send queue 상태입니다. send 대기중인 패킷이 거의 없는것으로 보아 네트워크 상태가 안정적으로 보입니다.
- **Contents Tick Duration p99**  
  게임서버에서 컨텐츠 스레드를 1번 업데이트 하는데 걸리는 시간이 약 25ms로 안정적으로 유지되었습니다.
- **DB Latency p99**  
  DB Latency는 약 3ms 로 안정적으로 유지되었습니다.
- **Stage Runtime Objects**  
  게임서버 1대당 몬스터의 수는 대략 3300마리 정도였습니다.
- **Host / Process CPU Usage**  
  게이트웨이서버의 CPU 사용률은 약 20%, 게임서버의 CPU 사용률은 약 2% 정도로 안정적으로 유지되었습니다. 나머지 서버들의 CPU 사용률은 미미합니다.
- **Host / Process Memory**  
  게이트웨이서버의 메모리 사용량은 약 60 MB, 게임서버의 메모리 사용량은 약 46 MB 로 안정적으로 유지되었습니다.


![drawio](/assets/img/posts/2026-07-29/DummyClient.PNG)
_6시간 돌린 후의 더미클라이언트 스크린샷_
- 6시간 동안 64,244,803 개의 패킷을 send 하였고, 822,176,603 개의 패킷을 recv 하였습니다. 
- 연결이 강제로 끊기는 등의 특별한 오류가 감지되지 않았습니다.
