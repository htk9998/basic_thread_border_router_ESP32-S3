# basic_thread_border_router_ESP32-S3 — ESP Thread Border Router 호스트 펌웨어

[ESP Thread Border Router 보드](https://docs.espressif.com/projects/esp-thread-br/en/latest/hardware_platforms.html)
(ESP32-S3-WROOM-1 8MB 호스트 + ESP32-H2-MINI-1 RCP 일체형)의 **S3 호스트** 펌웨어.
Thread 스택·보더라우터 기능(SRP 서버, mDNS 대리광고, NAT64)·Wi-Fi 백본·웹 UI·RCP 자동 업데이트를 담당한다.

원본: [esp-thread-br/examples/basic_thread_border_router](https://github.com/espressif/esp-thread-br/tree/main/examples/basic_thread_border_router)
(`README_upstream.md` 참조). 업스트림 트리 밖에서 빌드되도록 컴포넌트 경로만 `${ESP_THREAD_BR_PATH}` 환경변수로 바꿨다.

## 보드 구성

```
[Thread 기기들] ~~802.15.4~~ [ESP32-H2: ot_rcp] ──UART 460800 (Spinel)── [ESP32-S3: 이 펌웨어] ──Wi-Fi── LAN
                              ↑ RCP 펌웨어 저장소:                            웹 UI http://<BR-IP>
                              https://github.com/htk9998/ot_rcp_ESP32-H2
```

| 칩 | 역할 | 저장소 |
|---|---|---|
| ESP32-S3 | 호스트 — Thread 스택 + BR 기능 | **이 저장소** |
| ESP32-H2 | RCP — 802.15.4 송수신 전담 | https://github.com/htk9998/ot_rcp_ESP32-H2 |

`CONFIG_AUTO_UPDATE_RCP=y`(기본) — RCP 이미지가 이 펌웨어에 포함되어 첫 부팅 시 S3가 H2에 자동 플래시한다.
H2를 직접 굽는 작업은 불필요하며, **USB2(S3 포트)** 만 연결하면 된다.

## 빌드 환경

- ESP-IDF **v5.5.4** (esp-thread-br main 권장 버전)
- esp-thread-br 저장소 (컴포넌트 제공) → `ESP_THREAD_BR_PATH`로 지정
- RCP 이미지: 먼저 [ot_rcp_ESP32-H2](https://github.com/htk9998/ot_rcp_ESP32-H2)를 H2 타깃으로 빌드해두면 자동 포함됨

```bash
export ESP_THREAD_BR_PATH=~/esp/esp-thread-br
export OT_RCP_BUILD_DIR=~/esp/projects/ot_rcp_ESP32-H2/build   # RCP 산출물 위치 (sdkconfig CONFIG_RCP_SRC_DIR)
. ~/esp/esp-idf-v5.5.4/export.sh

# 1) RCP 먼저 (ot_rcp_ESP32-H2 저장소)
cd ~/esp/projects/ot_rcp_ESP32-H2 && idf.py set-target esp32h2 build

# 2) BR 호스트
cd ~/esp/projects/basic_thread_border_router_ESP32-S3
idf.py set-target esp32s3 build
idf.py -p /dev/ttyACM0 flash monitor      # USB2 = S3 포트
```

## 설정 포인트 (menuconfig)

| 항목 | 위치 | 기본 |
|---|---|---|
| Wi-Fi SSID/PW | Example Connection Configuration | 미설정 → 콘솔 `wifi -s SSID -p PW` 명령으로 지정 가능 |
| 자동 시작 | ESP Thread Border Router Example → automatic start mode | OFF (수동: 콘솔/웹 UI에서 `dataset init new` → `thread start`) |
| RCP 자동 업데이트 | AUTO_UPDATE_RCP | ON |
| RCP 이미지 경로 | RCP_SRC_DIR | `$ENV{OT_RCP_BUILD_DIR}` — 기본값(`$IDF_PATH/examples/openthread/ot_rcp/build`) 대신 별도 저장소 사용 |
| LWIP_IPV6_NUM_ADDRESSES | IDF v5.3.1+ | **12** |

## 동작 확인

부팅 로그에 `RCP Version in storage` / `Running RCP Version`이 같은 값으로 찍히면 RCP 업데이트 완료,
`OpenThread attached to netif` 이후 `SrpServer: Selected port` 가 보이면 BR 동작 중.
웹 UI(`http://<BR-IP>`)에서 Thread 네트워크 형성·Dataset 확인, 이후 chip-tool로 Matter 기기 커미셔닝:

```bash
chip-tool pairing ble-thread 0x7283 hex:<dataset> 20202021 3840
```

## 관련 저장소

- RCP 펌웨어: https://github.com/htk9998/ot_rcp_ESP32-H2
- Matter over Thread 테스트 디바이스/문서: https://github.com/htk9998/thread-on-matter-otbr
