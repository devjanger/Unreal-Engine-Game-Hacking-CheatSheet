# Unreal Engine Game Hacking CheatSheet

언리얼 엔진 게임 리버싱/진단 작업 시 참고용 명령어 모음.

## Table of Contents
- [1. Pak / Asset Extraction (repak)](#1-pak--asset-extraction-repak)
- [2. SDK Dumping (Dumper-7)](#2-sdk-dumping-dumper-7)
- [3. Offset Hunting (GSpots, SDK grep)](#3-offset-hunting-gspots-sdk-grep)
- [4. UEDumper Setup](#4-uedumper-setup)
- [5. Pointer Chains (UE4/UE5 Common)](#5-pointer-chains-ue4ue5-common)
- [6. Memory Editing (Cheat Engine)](#6-memory-editing-cheat-engine)
- [7. RPC Hooking (Frida)](#7-rpc-hooking-frida)
- [8. Asset Parsing (CUE4Parse)](#8-asset-parsing-cue4parse)
- [9. Network Diagnosis](#9-network-diagnosis)

---

## 1. Pak / Asset Extraction (repak)

[repak](https://github.com/trumank/repak) — pak 파일 패킹/언패킹 전용 도구.

### List contents of all .pak files
```powershell
Get-ChildItem *.pak | ForEach-Object {
    "===== $($_.Name) =====" | Out-File pak-list.txt -Append
    .\repak.exe list $_.FullName | Out-File pak-list.txt -Append
}
```

### Unpack all .pak files
```powershell
mkdir ".\output"
$cwd = (Get-Location).Path

Get-ChildItem *.pak | ForEach-Object {
    $outDir = "$cwd\output\$($_.BaseName)"
    .\repak.exe unpack $_.FullName -o $outDir
}
```

### Pack a mod (for distribution)
```powershell
.\repak.exe pack .\mod\ .\output_P.pak
```
> 파일명 끝에 `_P`가 없으면 게임이 원본을 우선 로드함. 적용 경로: `<Game>/Content/Paks/~mods/`

---

## 2. SDK Dumping (Dumper-7)

[Dumper-7](https://github.com/Encryqed/Dumper-7) — 클래스/구조체/함수/오프셋 전체를 C++ SDK로 덤프.

### Injection
```
1. DLL Injector 준비 (Xenos 등)
2. Process → 대상 게임 프로세스 선택
3. Add → Dumper-7.dll
4. Inject
```

### Output structure
```
C:\Dumper-7\<UE버전>-<게임이름>\
├── CppSDK\SDK\*.hpp/.cpp   ← 클래스/구조체/함수 + 오프셋 주석
├── Mappings\*.usmap         ← FModel / UAssetGUI용 매핑 파일
├── Dumpspace\*.json         ← JSON 형태 오프셋 (Classes/Structs/Enums/Functions/Offsets)
├── IDAMappings\*.idmap      ← IDA Pro용
└── GObjects-Dump*.txt
```

---

## 3. Offset Hunting (GSpots, SDK grep)

### GSpots (GOffsets 자동 탐색 도구 https://github.com/Do0ks/GSpots)
게임 실행 중 게임 파일을 `GSpots.exe`으로 드래그 앤 드랍 → GWorld / GNames / GObjects 오프셋 자동 출력. (오탐 주의)

### grep으로 SDK에서 멤버/함수 찾기
```powershell
# 멤버 변수 검색
Select-String -Path "Pal_classes.hpp" -Pattern "MaxWalkSpeed|MaxHP|CurrentHP"

# 함수 검색 (서버 RPC 후보 포함)
Select-String -Path "Pal_functions.cpp" -Pattern "ProcessDamage|ApplyDamage|_ToServer"

# 클래스 정의 위치
Select-String -Path "Engine_classes.hpp" -Pattern "class UCharacterMovementComponent"
```

Linux/WSL:
```bash
grep -n "MaxWalkSpeed\|MaxHP\|CurrentHP" Pal_classes.hpp
```

---

## 4. UEDumper Setup

[UEDumper](https://github.com/Spuckwaffel/UEDumper) — 실시간 메모리 인스펙터 + SDK 덤프 + Live Editor.

### Offsets.h
```cpp
offsets.push_back({ OFFSET_ADDRESS | OFFSET_DS, "OFFSET_GNAMES", 0x8E73E78 });
offsets.push_back({ OFFSET_ADDRESS | OFFSET_DS, "OFFSET_GOBJECTS", 0x8F6E910 });
offsets.push_back({ OFFSET_ADDRESS | OFFSET_DS | OFFSET_LIVE_EDITOR, "OFFSET_GWORLD", 0x90DD130 });
```

### UEdefinitions.h 체크리스트

| 항목 | 기본값 | 문제 시 시도 |
|---|---|---|
| `UE_VERSION` | 게임 버전에 맞게 | - |
| `WITH_CASE_PRESERVING_NAME` | FALSE | FNames 실패하면 TRUE |
| `BREAK_IF_INVALID_NAME` | TRUE | 디버깅 중엔 FALSE로 우회 |
| `FUOBJECTITEM_SIZE` | 24 (0x18) | 바이너리에서 `* 0x18` 직접 확인 |
| `UE_BLUEPRINT_EVENTGRAPH_FASTCALLS` | TRUE | 함수 오프셋 0이면 FALSE |

### 증상별 원인

| 증상 | 원인 후보 |
|---|---|
| `Caching Packages` 0%에서 멈춤 | GNames 오프셋/계산 방식 불일치 |
| `first object should be /Script/CoreUObject` 실패 | GNames 틀림 |
| `Could not find requested Pointer in cache` | `FUOBJECTITEM_SIZE` 불일치 |
| `No UStruct/Enum objects found` | GNames가 모두 빈 문자열로 캐싱됨 |

---

## 5. Pointer Chains (UE4/UE5 Common)

표준 패턴 — 오프셋은 게임/버전마다 다르므로 항상 SDK에서 재확인할 것.

```
base + GWORLD_OFFSET           → GWorld
GWorld + 0x158                 → GameState
GameState + 0x02A8             → PlayerArray (TArray<APlayerState*>)
PlayerArray[0]                 → PlayerState
PlayerState + PAWN_OFFSET      → Pawn (내 캐릭터, 예: 0x0308)
Pawn + 0x0320                  → CharacterMovement
CharacterMovement + 0x01F8     → MaxWalkSpeed (float)
```

### ULevel → Actors 배열 (버전에 따라 구조 다름)
```
ULevel + 0xE0       → ActorCluster (ULevelActorContainer*)  — null이면 맵 미입장 상태
ActorCluster + 0x28 → Actors (TArray<AActor*>)
```

### Python 예시 (pymem)
```python
import pymem, struct

pm = pymem.Pymem("Game-Win64-Shipping.exe")
base = pymem.process.module_from_name(pm.process_handle, "Game-Win64-Shipping.exe").lpBaseOfDll

def read_ptr(addr): return pm.read_longlong(addr)
def read_float(addr): return pm.read_float(addr)
def write_float(addr, val): pm.write_float(addr, val)

gworld = read_ptr(base + 0x90DD130)
gamestate = read_ptr(gworld + 0x158)
player_array = read_ptr(gamestate + 0x2A8)
player_state = read_ptr(player_array)
pawn = read_ptr(player_state + 0x308)
movement = read_ptr(pawn + 0x320)

write_float(movement + 0x1F8, 3000.0)  # MaxWalkSpeed 변경
```

---

## 6. Memory Editing (Cheat Engine)

### MemRE - 언리얼 엔진 게임 전용 메모리 편집기(치트엔진)
https://github.com/Do0ks/MemRE

<img width="1563" height="900" alt="image" src="https://github.com/user-attachments/assets/30305c24-718e-4bbc-ae8a-9f5e18706bc4" />


### Pointer Scan으로 동적 오프셋 찾기
게임 재시작마다 주소가 바뀌어서 오프셋 체인을 매번 다시 찾아야 할 때, Cheat Engine의 Pointer Scan 기능을 쓰면 자동화할 수 있다.
1. 현재 값 주소를 일반 스캔으로 찾기 (예: HP, 좌표값)
2. 해당 주소 우클릭 → "Pointer scan for this address"
3. Max level: 5~7, Max offset: 4096 정도로 설정 후 스캔
4. 게임 재시작 → 같은 포인터 스캔 결과(.PTR 파일)를 재오픈
5. "Rescan memory" → 재시작 후에도 유효한 포인터 체인만 필터링되어 남음
재시작 후에도 살아남는 포인터 체인이 진짜 안정적인 오프셋. 여러 번 재시작 후 교집합을 구하면 신뢰도가 더 올라간다.


### Lua 스크립트로 자동화 (CEA Lua)
치트테이블(.CT) 자체에 Lua 스크립트를 심어서, 테이블을 열면 자동으로 포인터 체인을 따라가고 값을 표시/수정하는 GUI 폼을 만들 수 있다.
``` lua
luafunction getPlayerHP()
    local base = getAddress("게임exe이름.exe")
    local gworld = readPointer(base + 0x90DD130)
    local gamestate = readPointer(gworld + 0x158)
    local playerArray = readPointer(gamestate + 0x2A8)
    local playerState = readPointer(playerArray)
    local pawn = readPointer(playerState + 0x308)
    return readFloat(pawn + 0x???)  -- HP 오프셋
end

createTimer(getMainForm(), 1000, function()
    print("Current HP: " .. tostring(getPlayerHP()))
end)
```
이러면 치트엔진 안에서 .CT 파일 하나로 배포할 수 있다. 작업 때 만든 포인터 체인을 그대로 Lua 문법으로 옮기면 됨.

---

## 7. RPC Hooking (Frida)

서버 검증 여부 테스트용. 함수 이름에 `_ToServer`, `Server*`, `*RPC*` 패턴이 있으면 클라이언트→서버 호출 함수.

```javascript
const base = Module.getBaseAddress("Game-Win64-Shipping.exe");
const funcAddr = base.add(0x함수오프셋); // SDK의 *_functions.cpp에서 찾기

Interceptor.attach(funcAddr, {
    onEnter: function(args) {
        console.log("서버로 전송되는 값:", args[1]);
        // args[n]을 수정하면 조작된 값이 그대로 서버에 전송됨
    }
});
```

---

## 8. Asset Parsing (CUE4Parse)

[CUE4Parse](https://github.com/FabianFG/CUE4Parse) — FModel이 내부적으로 쓰는 것과 동일한 파싱 엔진. C# 코드로 직접 호출 가능.

```powershell
dotnet new console
dotnet add package CUE4Parse
```

```csharp
using CUE4Parse.FileProvider;
using CUE4Parse.UE4.Versions;
using CUE4Parse.UE4.Objects.Core.Misc;
using CUE4Parse.Encryption.Aes;

var provider = new DefaultFileProvider(
    @"C:\...\Content\Paks",
    SearchOption.AllDirectories, true,
    new VersionContainer(EGame.GAME_UE5_1));

provider.Initialize();
provider.SubmitKey(new FGuid(), new FAesKey(new byte[32])); // 비암호화 게임용

// usmap 적용 (선택)
provider.MappingsContainer = new CUE4Parse.MappingsProvider.FileUsmapTypeMappingsProvider(usmapPath);

// 경로 검색
var matches = provider.Files.Keys.Where(k => k.Contains("Grenade"));

// 오브젝트 JSON 추출
var obj = provider.LoadObject("Pal/Content/.../AB_Grenade_C.AB_Grenade_C");
Console.WriteLine(JsonConvert.SerializeObject(obj, Formatting.Indented));
```

### 관련 기존 도구

| 도구 | 용도 |
|---|---|
| [FModel](https://github.com/4sval/FModel) | GUI 에셋 탐색기 (CUE4Parse 기반) |
| [CUE4Parse.CLI](https://github.com/joric/CUE4Parse.CLI) | CUE4Parse 공식 CLI (fork) |
| [UAssetAPI](https://github.com/atenfyr/UAssetAPI) | uasset 직접 읽기/편집 라이브러리 |
| [MCP-UAsset-Toolkit](https://github.com/ShadowNineX/MCP-UAsset-Toolkit) | UAssetAPI 기반 MCP 서버 (Claude 등에서 직접 호출 가능) |

---

## 9. Network Diagnosis

### 호스트 권한 구조 판단
```
호스트가 곧 서버 (리슨 서버)        → 자신의 메모리 조작 테스트는 검증 의미 없음
  예: 4인 코옵 호스팅, Listen Server

별도 데디케이드 서버 프로세스 존재    → 의미 있는 검증 환경
  예: PalServer.exe + 별도 클라이언트로 접속
```

### 판단 기준
```
"내가 서버 권한을 가진 상태에서 테스트하는가?"
  Yes → 클라이언트 조작 테스트 자체가 무효 (자기 자신을 속이는 것)
  No  → 서버 측 검증 여부를 제대로 테스트하는 것
```

### P2P 환경의 한계
- Steam Networking Sockets는 기본 암호화 → Wireshark로 페이로드 직접 해석 불가
- 트래픽이 Steam 릴레이를 거칠 수 있어 경로 자체가 불확실
- 패킷 캡처보다 RPC 함수 후킹(섹션 7)이 훨씬 실용적

---

## Workflow Summary

```
1. GSpots로 1차 오프셋 확인
2. Dumper-7 DLL 인젝션 → SDK 전체 덤프
3. SDK(.hpp)에서 필요한 클래스/오프셋 grep
4. (선택) UEDumper Live Editor로 실제 메모리 값 검증
5. Python(pymem) 또는 Frida로 포인터 체인 코드 작성
6. 실행 → 결과 확인 → 오프셋 안 맞으면 SDK 재확인 후 수정
7. (네트워크 진단 시) RPC 함수 후킹으로 서버 전송값 가로채기
8. (필요 시) 데디케이드 서버 환경에서 클라-서버 분리 테스트
```

---

## Disclaimer
연구/진단 목적으로만 사용. 온라인 멀티플레이 환경에서의 무단 메모리 조작은 서비스 약관 위반 및 법적 문제가 될 수 있음. 본인 소유 게임의 싱글플레이/오프라인 환경, 또는 정당한 권한이 있는 보안 진단 대상에서만 사용할 것.
