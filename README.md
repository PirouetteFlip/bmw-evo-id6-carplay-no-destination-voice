# bmw-evo-id6-carplay-no-destination-voice
Restore CarPlay navigation voice guidance when driving without a destination on BMW EVO iDrive 6 (NBT EVO ID6) via a 1-instruction patch

English: [README.en.md](README.en.md)



# BMW EVO ID6 CarPlay 안심주행 안내음 패치 정리

Nov 25, 2026 · PirouetteFlip

**라이선스**: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ko) © 2026 PirouetteFlip. 출처(작성자와 원문 링크)를 표기하면 누구나 자유롭게 복사, 수정, 재배포할 수 있습니다. 아래 위험 안내와 책임 고지는 그대로 적용됩니다.

## 위험 안내와 책임 고지

- **부팅 불가 위험**: 헤드유닛 실행 파일을 수정하는 일이라 잘못하면 부팅이 안 될 수 있습니다.
- **백업 먼저**: 백업을 두 곳(차 내부 파일시스템 + USB)에 만든 뒤 진행하세요.
- **책임**: 모든 책임은 본인에게 있으며, 차량 보증에 영향을 줄 수 있습니다.
- **작성자의 범위**: 작성자는 본인 차량 한 대에 적용하고 검증한 값을 공유할 뿐이며, 책임을 지거나 문제가 생겼을 때 도와드릴 수 없습니다.
- **다른 펌웨어/차종**: 헤드유닛 내부 실행 파일 구조가 다를 수 있으니, AI 등을 이용해 본인 차량에 적용할 수 있는지 확인한 뒤 시도하시길 권합니다.
- **공유 범위**: 공유하는 내용은 패치 프로그램이 아니라 적용 방법에 대한 설명뿐입니다.

## 요약

목적지 없이 달리는 안심주행에서 CarPlay 내비 안내음이 안 나던 문제를, `Connectivity` 실행 파일의 명령어 4바이트를 바꿔 해결했고 한 대의 실차에서 동작을 확인했습니다.

- **증상**: 안심주행(목적지 없이 주행)에서 Tmap, 네이버지도, 애플지도 모두 음성 안내가 나오지 않습니다.
- **원인**: 목적지를 설정하면 내비 권한이 아이폰으로 넘어가고(CarPlay 로그의 `Navigation Owner: Apple Device`), 그때만 CarPlay 안내음이 출력 채널로 연결됩니다(source 23 → sink 4). 안심주행은 `Navigation Owner: None`이라 안내음이 전달되지 않습니다.
- **해결**: `handleAltAudioUpdate` 안에서 길안내 중인지 확인하는 조건 분기 `beq` 1개를 NOP으로 교체합니다. 목적지 설정 여부와 상관없이 안내음이 전달되어 안심주행에서도 소리가 납니다.

## 테스트 환경

주의! 아래 조합에서만 확인했고, 다른 펌웨어에서는 같은 바이트 위치가 아닐 수 있습니다.

| 항목 | 내용 |
| --- | --- |
| 차량 | BMW F33 LCI (한국 마켓) |
| 헤드유닛 | EVO iDrive 6 (NBT EVO ID6), QNX 기반, 계기판 6WB |
| 접속 | OBD-ENET 케이블 + 이더넷 어댑터, SSH (쉘 프롬프트 `hu-omap`) |
| 대상 파일 | `/opt/conn/bin/Connectivity` (ARM ELF, 7,529,956 바이트) |
| 원본 SHA-256 | `17a31d3cabfb4d1d3ae535d85d4e1999ef999d57de3face258401d4396641d5f` |

## 간단한 설명

`handleAltAudioUpdate`는 CarPlay 안내음(Alt Audio) 상태가 바뀔 때 불리는 함수입니다.

목적지 없이 주행할 때는 안내음 경로 요청을 건너뜁니다.&#32;

아래는 디스어셈블리를 단순화한 설명입니다.

```
handleAltAudioUpdate(duckAudio, connectionID):
    로그 출력(bDuckAudio, mConnctionID, CountBuildHighSync, TurnByTurn)
    turnByTurn = CarPlay 길안내 상태      # 안심주행이면 0

    if turnByTurn == 0:                   # cmp r0,#0 ; beq  ← 패치 전: 여기서 점프
        return                            # 안내음 경로를 요청하지 않음 → 소리 없음

    requestBuildHighSyncConnection        # (안내음 경로) 0이 아니면 안내음 전달
                                          # → jacinto: source 23 → sink 4 연결
```

1. **비교** `cmp r0,#0`: 길안내 상태가 0(꺼짐)인지 확인합니다.
2. **점프** `beq` (`19 00 00 0a`): 0이면 연결 요청을 건너뜁니다. 안심주행에서 소리가 안 났던 이유입니다.
3. **연결 요청** `requestBuildHighSyncConnection`: 0이 아니면 안내음 경로(source 23 → sink 4)를 요청합니다.

✅ **수정할 부분**: `beq`를 `nop`(`00 00 a0 e1`)으로 바꾸면 점프가 없어져 3번이 상태와 상관없이 항상 실행됩니다.

**한계**: 의사코드는 이 분기 주변만 이해를 위해 간단하게 표현한 내용입니다. 실제 함수에는 덕/언덕 구분과 연결 횟수(`CountBuildHighSync`) 처리 등 다른 분기가 더 있고, 전부 분석하지는 않았습니다. 그래서 위 ‘부작용 가능성’이 남아 있습니다.

## 적용과 복원

`Connectivity` 파일 안의 4바이트 내용을 변경합니다. 실행 중인 `Connectivity` 프로세스의 메모리는 직접 쓰기가 막혀 있어, 원본을 복사해 수정한 파일로 교체하고 재부팅해서 적용합니다.

파일이 들어 있는 `/fs/sda0`(`/opt`)는 기본이 **읽기 전용**입니다. 그래서 교체 직전에 `mount -uw /fs/sda0`로 쓰기 가능으로 바꾸고, 파일을 바꾼 뒤 `mount -ur /fs/sda0`로 다시 읽기 전용으로 되돌립니다. 이 과정은 아래 적용 순서에 포함되어 있습니다.

| 항목 | 값 |
| --- | --- |
| 위치 | `dd bs=4` 기준 블록 1306899 |
| 원본 | `19 00 00 0a` (`beq`) |
| 패치 | `00 00 a0 e1` (NOP) |
| 주변 20바이트(원본) | `2c 30 93 e5  33 ff 2f e1  00 00 50 e3  19 00 00 0a  24 30 94 e5` |

차량 쉘에는 `od`, `hexdump`, `xxd`, `touch`, `chown`, `awk`, `python` 등이 없습니다. 바이트 확인은 `sed -n l`로 했고, 바이너리 문자열은 ksh 내장 `print -n`으로 만들었습니다.

### 적용 순서

**사전 준비: SSH 접속**

이 방법은 헤드유닛에 SSH로 접속할 수 있어야 쓸 수 있습니다. SSH는 기본으로 열려 있지 않아서, 기존에 알려진 방법으로 먼저 열어야 합니다. 이 문서는 SSH를 여는 방법 자체는 다루지 않습니다.

- **연결**: OBD-ENET 케이블과 이더넷 어댑터로 단말과 헤드유닛을 이어 SSH로 접속합니다. 작성자는 안드로이드 폰의 터미널 앱을 써서 접속했습니다.
- **접속 확인**: 쉘 프롬프트가 `hu-omap:/dev/shmem>`처럼 보이면 omap 쪽 쉘에 들어온 것입니다. 아래 명령은 모두 이 쉘에서 실행합니다.
- **진행 조건**: 정차 상태에서, 아래 1단계 백업을 끝내기 전에는 파일을 건드리지 마세요.

**준비물: 백업용 USB**

원본 `Connectivity`를 차 밖에 한 번 더 보관하려면 쓰기가 되는 USB 메모리를 차의 USB 포트에 꽂아 두어야 합니다. 파일이 약 7.5MB라 여유 공간이 8MB 이상이면 됩니다. 아래 1단계를 시작하기 전에 SSH로 접속해 `ls /fs/usb0`가 되는지 확인하세요.

**사전 점검: 원본 바이트 확인**

패치하기 전에 대상 위치가 이 문서의 원본과 같은지 먼저 확인합니다. 아래 출력이 다르면 다른 펌웨어이므로 여기서 멈추세요.

```
dd if=/opt/conn/bin/Connectivity bs=4 skip=1306896 count=5 | sed -n l
# 기대 출력(원본, 위 표의 바이트로 계산한 값): 두 줄로 나뉘어 보입니다
# ,0\223\3453\377/\341\000\000P\343\031\000\000$
# $0\224\345$
```

1. **백업 (원본 2곳)**. USB에 `t_Connectivity.orig`를 만들고, 차 내부 백업(`Connectivity.orig`)은 5단계 교체 과정에서 만들어집니다.

```
cp /opt/conn/bin/Connectivity /fs/usb0/t_Connectivity.orig
ls -l /fs/usb0/t_Connectivity.orig   # 7529956 바이트인지 확인
```

2. **NOP 4바이트 파일 만들기** (`/dev/shmem`은 RAM이라 시동을 끄면 사라집니다).

```
(dd if=/dev/zero bs=1 count=2 2>/dev/null; print -n '\0240\0341') > /dev/shmem/t_nop
```

3. **RAM에서 복사본을 패치**. 라이브 파일은 건드리지 않습니다. 목적은 정확히 4바이트만 덮어쓰는 것이고, 아래 `dd` 옵션은 작성자가 쓴 방식을 정리한 것입니다. 끝난 뒤 파일 크기가 7,529,956 바이트 그대로인지 반드시 확인하고, 다르면 중단하세요.

```
cp /opt/conn/bin/Connectivity /dev/shmem/t_Conn_patched
dd if=/dev/shmem/t_nop of=/dev/shmem/t_Conn_patched bs=4 seek=1306899 conv=notrunc
ls -l /dev/shmem/t_Conn_patched
```

4. **패치 확인**. 패치 후에는 다음처럼 한 줄로 출력되어야 합니다(원본은 `0x0a` 때문에 줄이 나뉘어 보입니다). 라이브 파일은 아직 원본이어야 합니다.

```
dd if=/dev/shmem/t_Conn_patched bs=4 skip=1306896 count=5 | sed -n l
# 기대 출력: ,0\223\3453\377/\341\000\000P\343\000\000\240\341$0\224\345$
```

5. **교체**. 실패하면 이 순서를 중단하고 원본을 되살리세요. `/fs/sda0`은 기본이 읽기 전용이라 잠깐 쓰기 가능으로 바꾸고, 끝나면 다시 읽기 전용으로 돌립니다.

```
mount -uw /fs/sda0
cp /dev/shmem/t_Conn_patched /opt/conn/bin/Connectivity.new && chmod 775 /opt/conn/bin/Connectivity.new
mv /opt/conn/bin/Connectivity /opt/conn/bin/Connectivity.orig && mv /opt/conn/bin/Connectivity.new /opt/conn/bin/Connectivity
dd if=/opt/conn/bin/Connectivity bs=4 skip=1306896 count=5 | sed -n l
mount -ur /fs/sda0
```

6. **헤드유닛 재부팅** (재부팅하면 SSH 연결이 끊기므로 부팅이 끝난 뒤 다시 접속). `Connectivity`는 죽어도 자동 재시작되지 않으니(`MaxProcessRestarts=0`) 재부팅으로 적용하는 편이 안전합니다.
7. **확인**. 목적지 없이 CarPlay를 띄우고 안내음이 나는지 보고, 음악, 전화, 순정 네비도 같이 확인합니다. 로그로는 소리 직후 10초 안에 `on -f hu-jacinto sloginfo | grep "sourceID=23"`를 보면 됩니다(로그가 빨리 밀립니다).

### 복원

```
mount -uw /fs/sda0
cp /opt/conn/bin/Connectivity.orig /opt/conn/bin/Connectivity.rst && mv /opt/conn/bin/Connectivity.rst /opt/conn/bin/Connectivity
mount -ur /fs/sda0
```

복원 후 헤드유닛을 재부팅합니다. `Connectivity`는 SSH 서버를 관리하지 않아(프로세스 관리 설정에 SSH 항목이 없음) 이 파일에 문제가 생겨도 SSH는 살아 있을 것으로 보지만, 이 부분은 실제로 시험하진 않았습니다. USB의 `t_Connectivity.orig`가 마지막 백업입니다.

### 문제가 생겼을 때

- **5단계 도중 오류**: `Connectivity`가 없거나 크기가 이상하면 쓰기 가능 상태에서 `mv /opt/conn/bin/Connectivity.orig /opt/conn/bin/Connectivity`로 되돌린 뒤 `mount -ur /fs/sda0`를 실행합니다. 남은 `Connectivity.new`는 무시해도 됩니다.
- **재부팅 후 CarPlay나 소리가 이상하고 SSH는 접속됨**: 위 ‘복원’ 절차를 실행하고 다시 재부팅합니다.
- **재부팅 후 SSH까지 안 됨**: 이 문서로는 복구할 수 없습니다. 작성자는 이런 상황을 겪지 않았고 복구 방법도 검증하지 못했습니다. 그래서 정차 상태에서, 5단계 확인 출력이 맞는 것을 본 뒤에만 재부팅하세요.

## 확인된 것

로그 비교, 바이너리 역분석, 실차 테스트 세 가지로 확인한 내용입니다.

- **로그 비교**: 목적지를 설정한 로그에서는 jacinto에 `sourceID=23 → sinkID=4` 연결이 생기고, 안심주행 로그에서는 한 번도 생기지 않습니다. 순정 네비 음성(`sourceID=17 → sinkID=4`)은 항상 연결됩니다.
- **Navigation Owner 로그 출처**: CarPlay 연결 라이브러리(`libconn_carplay.so`)가 omap 쪽 로그에 `Modes changed: … Navigation Owner : Apple Device` 또는 `… Navigation Owner : None` 형태로 남깁니다. 목적지를 설정했을 때와 안심주행일 때의 이 줄을 비교하면 같은 증상인지 확인할 수 있습니다.
- **오디오 스트림은 도착함**: 안심주행에서도 CarPlay Alt Audio(덕 오디오) 스트림은 헤드유닛까지 오지만 출력 연결이 없어 소리가 나지 않습니다.
- **조건 분기 위치**: `Connectivity`의 `handleAltAudioUpdate`에서 TurnByTurn 값이 0이면 `requestBuildHighSyncConnection` 호출을 건너뛰는 `cmp r0,#0` + `beq` 쌍이 있습니다. 파일 오프셋 `0x4FC44C`이 해당 `beq`입니다.
- **실차 테스트**: 패치 후 재부팅하고 목적지 없이 주행하면 안내음이 나옵니다. 음악, 전화, 순정 네비도 정상입니다. 재부팅 후에도 패치는 유지됩니다(파일 자체를 바꿨기 때문).
- **영향 없었던 시도**: `carplay.ini`의 `NativeNaviSupported=NO`는 효과가 없었고, `FormatSupport`의 AltAudio 줄을 지우면 CarPlay 소리 자체가 망가졌습니다. 두 가지 모두 원래대로 되돌렸습니다.

## 주의사항

- **다른 펌웨어/차종**: 오프셋과 바이트는 위 SHA-256 파일 기준입니다. 해시가 다르면 그대로 쓰지 말고, 위 ‘적용과 복원’ 표의 원본 바이트가 맞는지 먼저 확인하세요.
- **부작용 가능성**: 안내음 경로를 항상 요청하므로 전화, Siri, 음악 복귀 등 다른 상황에서 소리가 겹치거나 끊기는지는 장기간 확인하지 못했습니다. 목적지를 설정한 경우의 동작도 장기 테스트 전입니다.
- **미설명 로그**: 목적지 없을 때만 `kDuplicateErr` AirPlay assert가 보였습니다. 패치와의 관계는 조사하지 않았습니다.
- **소유자 권한**: 차에 `chown`이 없어 파일 소유자가 root로 바뀌었습니다(권한은 775 유지). 문제는 관찰되지 않았습니다.
- **업데이트**: 펌웨어 업데이트나 코딩으로 `Connectivity`가 교체되면 패치가 사라집니다.

## 수정 이력

| 날짜 | 내용 |
| --- | --- |
| 2026-11-25 | 1차 작성. BMW F33 LCI 한 대에서 검증 |
