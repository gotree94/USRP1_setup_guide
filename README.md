# Ettus Research USRP1 구동 가이드 (Windows 기준)

> 대상 하드웨어: Ettus Research USRP1 마더보드
> (참고 이미지: [ResearchGate – USRP-1 motherboard](https://www.researchgate.net/figure/USRP-1-motherboard_fig2_319174918))
>
> 본 문서는 **환경설정(드라이버/소프트웨어 설치)부터 실제 예제 실행까지** USRP1 보드에 맞게 정리한 가이드입니다.

---

## 목차

1. [USRP1 하드웨어 개요](#1-usrp1-하드웨어-개요)
2. [필요한 준비물](#2-필요한-준비물)
3. [소프트웨어 설치](#3-소프트웨어-설치)
4. [하드웨어 연결 및 확인](#4-하드웨어-연결-및-확인)
5. [장치 인식 확인](#5-장치-인식-확인)
6. [예제 실행 (USRP1에 맞는 파라미터)](#6-예제-실행-usrp1에-맞는-파라미터)
7. [파라미터 해설 (USRP1 기준)](#7-파라미터-해설-usrp1-기준)
8. [문제 해결 (트러블슈팅)](#8-문제-해결-트러블슈팅)
9. [USRP1의 제약 사항과 고급 팁](#9-usrp1의-제약-사항과-고급-팁)
10. [참고 자료](#10-참고-자료)

---

## 1. USRP1 하드웨어 개요

USRP1은 Ettus Research가 출시한 **최초의 범용 소프트웨어 라디오(SDR)** 로, 사진의 파란색 마더보드가 바로 USRP1입니다.

### 1.1 주요 사양

| 항목 | 내용 |
| --- | --- |
| 호스트 인터페이스 | **USB 2.0 High-Speed** (480 Mb/s, Cypress FX2 CY7C68013A) |
| FPGA | Altera Cyclone EP1C12 |
| ADC | 4채널, **12-bit / 64 MS/s** |
| DAC | 4채널, **14-bit / 128 MS/s** |
| DSP | FPGA 내부 DDC 2개(RX) + DUC 2개(TX) |
| 클록 | 64 MHz 고정 (master clock rate = 64 MHz) |
| 데몬보드 슬롯 | **2개** (Slot A, Slot B) — 2×2 MIMO 가능 |
| RF 대역폭 | 최대 **8 MHz** (16-bit 샘플, sc16) / **16 MHz** (8-bit 샘플, sc8, RX 전용) |
| USB 최대 데이터율 | 32 MB/s (`USRP1_MAX_RATE_USB2 = 32000000 bytes/s`) |
| USB VID:PID | `0xfffe:0x0002` |
| 전원 | 외부 어댑터 (5.9V / 4A, 2.1mm/5.5mm DC 커넥터) |

### 1.2 보드 구성 요소 (사진 기준)

- **FX2 USB 인터페이스 칩**(Cypress CY7C68013A): PC와 USB 통신 담당. 전원을 넣으면 먼저 이 칩의 펌웨어(`usrp1_fw.ihx`)가 로드되어야 동작합니다.
- **Altera Cyclone FPGA**: USB로 받은 데이터의 디지털 하향변환(DDC)/상향변환(DUC), 필터링 등을 수행. FPGA 이미지(`usrp1_fpga.rbf`)가 로드됩니다.
- **AD9862 CODEC 2개** (Slot A/B 각 1개): ADC/DAC 신호를 FPGA와 연결.
- **데몬보드 슬롯 2개 (A/B)**: RF 프론트엔드(송수신 카드)를 꽂는 위치. 좌우가 180도 반대 방향으로 장착됩니다.
- **6V/4A 전원 커넥터**, **USB-B 커넥터**, **LED**(전원/상태 표시).

### 1.3 동작 원리 (간단히)

```
안테나 ─ 데몬보드(RF) ─ ADC(64 MS/s) ─ [FPGA DDC/필터] ─ USB 2.0 ─ PC(UHD/GNU Radio)
                                    (12-bit)                       (sc16: I,Q 16bit×2)
```

- 수신(RX): 데몬보드가 RF 신호를 중간주파수로 내리고, ADC가 64 MS/s로 샘플링 → FPGA DDC가 **decimation(다운샘플)** 해서 USB로 내보냄.
- 송신(TX): PC의 샘플을 USB로 보내면 FPGA DUC가 **interpolation** 한 뒤 DAC(128 MS/s) → 데몬보드에서 RF로 변환.

> **중요:** USRP1은 **USB 2.0 전용**이며 USB 1.1에서는 동작하지 않습니다. USB 3.0 포트는 하위 호환으로 동작하지만, 안정성이 문제가 되면 별도의 USB 2.0(EHCI) 포트나 전원 공급 허브를 사용하세요.

---

## 2. 필요한 준비물

### 2.1 하드웨어

| 준비물 | 설명 |
| --- | --- |
| USRP1 마더보드 | 이미 소유 (사진 참조) |
| 전원 어댑터 | 6V급(예: Ettus P/N `PS6V4A`, 5.9V/4A), 2.1mm/5.5mm DC 커넥터 (USRP1 자체는 5V/2A면 충분하지만 데몬보드 전원을 위해 6V/4A 권장) |
| USB 2.0 케이블 | USB-B to USB-A, 가능하면 품질 좋고 짧은 케이블 |
| 데몬보드(Daughterboard) | **최소 1개 필요** (예: WBX, SBX, TVRX2, LFRX/LFTX, BasicRX/BasicTX 등). 마더보드만으로는 RF 송수신이 불가능합니다. |
| 안테나 / RF 케이블 | 데몬보드의 주파수 범위에 맞는 안테나와 SMA 케이블 |
| (옵션) 감쇠기 | 송신(TX) 테스트 시 수신기를 보호하기 위한 감쇠기 (예: Ettus LPBK-KIT, 30~40 dB) |
| (옵션) 전원 공급 USB 허브 | USB 전원/신호 안정화용 |

### 2.2 PC 요구 사항

- Windows 10 / 11 (64-bit) — UHD 4.9 기준 검증 환경: Windows 10 22H2, Windows 11 23H2
- USB 2.0(EHCI) 포트 또는 USB 3.0 호환 포트
- 인터넷 연결 (드라이버/이미지 다운로드용)
- 여유 디스크 공간 약 1~2 GB

---

## 3. 소프트웨어 설치

### 3.1 설치 전 알아둘 점

- USRP1은 오래된 기기이지만 **최신 UHD(4.x)에서도 공식 지원**됩니다.
  - 참고: UHD 4.9 릴리스에도 "USRP1" 관련 수정(property tree fix 등)이 포함되어 있습니다.
  - 최신 UHD 메뉴얼의 `page_usrp1.html`에 USRP1 문서가 계속 제공되고 있습니다.
- Windows에서 USRP1은 **WinUSB 드라이버 + libusb**로 동작합니다. 드라이버 미설치 시 "알 수 없는 장치"로 보입니다.
- USRP1은 USB로 데이터가 나오기 전에 **펌웨어 + FPGA 이미지 로딩**이 필요하며, 이는 UHD가 자동으로 수행합니다 (`usrp1_fw.ihx`, `usrp1_fpga.rbf`).
- 설치 순서 요약:
  1. UHD 설치 (또는 GNU Radio 번들)
  2. USB 드라이버(WinUSB) 설치
  3. UHD 이미지(펌웨어/FPGA) 내려받기
  4. 보드 연결 → `uhd_find_devices` / `uhd_usrp_probe`로 확인

### 3.2 경로 A: UHD 설치 (권장, CLI 중심)

Ettus가 제공하는 공식 Windows 바이너리 인스톨러를 사용합니다.

1. 최신 UHD 설치 파일 다운로드:
   - `http://files.ettus.com/binaries/uhd/latest_release`
   - 예: `uhd_4.9.0.1-release_Win64_VS2022.exe` 형식의 파일
2. 다운로드한 `.exe`를 실행하여 설치합니다.
   - 기본 설치 위치: `C:\Program Files\UHD`
3. (필요 시) MSVC 재배포 패키지(Visual C++ Redistributable)를 설치합니다. 최신 UHD 인스톨러는 보통 함께 처리됩니다.

설치 후 주요 구성 요소:

| 경로 | 내용 |
| --- | --- |
| `C:\Program Files\UHD\bin\` | `uhd_find_devices`, `uhd_usrp_probe`, `uhd_config_info` 등 실행 파일 |
| `C:\Program Files\UHD\lib\uhd\examples\` | C++ 예제 실행 파일 (`rx_samples_to_file.exe`, `tx_samples_from_file.exe` 등) |
| `C:\Program Files\UHD\lib\uhd\examples\python\` | Python 예제 (`rx_to_file.py` 등) |
| `C:\Program Files\UHD\lib\uhd\utils\uhd_images_downloader.py` | 이미지 다운로더 |
| `C:\Program Files\UHD\share\uhd\usbdriver\` | USB 드라이버 `.inf` 파일 (수동 설치 시 사용) |

### 3.3 USB 드라이버(WinUSB) 설치

- **UHD 4.9 이상**에서는 인스톨러가 **USB 드라이버를 자동 설치**하므로 보통 별도 작업이 필요 없습니다.
- 구버전 UHD 또는 자동 설치가 안 된 경우:
  1. 드라이버 패키지 다운로드: `http://files.ettus.com/binaries/misc/erllc_uhd_winusb_driver.zip` (압축 해제)
  2. USRP1을 USB에 연결 → **장치 관리자**에서 "기타 장치 > 알 수 없는 장치 / USRP" 항목을 우클릭
  3. **드라이버 업데이트 > 컴퓨터에서 드라이버 찾아보기** 선택
  4. `erllc_uhd_winusb_driver.zip` 압축 해제 폴더(또는 `C:\Program Files\UHD\share\uhd\usbdriver`)를 지정하고 `*.inf` 파일 선택 → 설치
- 장치 관리자에서 **"Ettus Research USRP"** 또는 **"USRP"** 이름으로 보이면 성공입니다.
- Windows 10/11에서 설치 마지막에 오류 메시지가 뜨는 경우가 있습니다. **알려진 이슈**이며, USRP를 재부팅(전원 껐다 켬)하면 정상 동작합니다.

> 팁: 재인식이 잘 안 되면 **장치 관리자에서 장치를 "제거"(드라이버 소프트웨어 삭제 포함) 후 USB 케이블을 뽑고 다시 연결**해보세요.

### 3.4 이미지(펌웨어 + FPGA) 내려받기

USRP1이 USB로 데이터를 주고받으려면 아래 이미지가 PC에 있어야 합니다.

- 펌웨어: `usrp1_fw.ihx`
- FPGA: `usrp1_fpga.rbf` (기본, 2×DDC + 2×DUC) / `usrp1_fpga_4rx.rbf` (옵션, 4×DDC)

명령 프롬프트(CMD) 또는 PowerShell에서:

```
uhd_images_downloader
```

- `uhd_images_downloader`가 PATH에 없으면 Python으로 직접 실행:

```
python "C:\Program Files\UHD\lib\uhd\utils\uhd_images_downloader.py"
```

- 이미지는 기본적으로 `%APPDATA%\uhd\images` (즉 `C:\Users\<사용자>\AppData\Roaming\uhd\images`)에 저장됩니다.
- 확인:

```
dir "%APPDATA%\uhd\images\usrp1_*"
```

`usrp1_fw.ihx`와 `usrp1_fpga.rbf`가 보이면 준비 완료입니다.

### 3.5 (선택) UHD Python API

Python에서 UHD를 직접 사용하려면 버전이 맞는 `uhd` wheel을 설치합니다.

```
"C:\Program Files\UHD\bin\uhd_config_info.exe" --version
python -m pip install uhd==<위에서 확인한 버전>
```

검증:

```
python -c "import uhd; print(uhd.USRP_SERIAL)")
```

> 주의: UHD 인스톨러와 wheel 버전은 반드시 일치해야 합니다.

### 3.6 경로 B: GNU Radio 설치 (GUI / GRC)

스펙트럼 분석(GUI)이나 GNU Radio Companion(GRC) 플로우그래프를 쓰려면 GNU Radio가 필요합니다. 아래 3가지 중 하나를 선택하세요.

| 방법 | 특징 | 다운로드 |
| --- | --- | --- |
| **RadioConda (권장)** | GNU Radio 위키 공식 권장. conda 기반, 최신 3.10.12.0, UHD 포함 | https://github.com/ryanvolz/radioconda (또는 radioconda-installer Releases) |
| **CASTLE 인스톨러** | 설치가 간편. GNU Radio 3.10.12.0 + **UHD 4.6.0.0** 번들 | https://www.castle.cloud/gnuradio-for-windows/download/ |
| **Ettus/NI 실험용 인스톨러** | GNU Radio 3.11 + **UHD 4.9**. 실험판이므로 GRC는 GTK 전용, matplotlib 미포함 등 알려진 제약 있음 | https://files.ettus.com/binaries/gnuradio/latest_stable/ |

**RadioConda 설치 절차:**
1. `radioconda-Windows-x86_64.exe` 다운로드 → 실행 (기본 경로 `C:\Users\<사용자>\radioconda`)
2. 시작 메뉴에서 **"GNU Radio Companion"** 실행 (GRC가 뜨면 성공)
3. **"radioconda Prompt"**(시작 메뉴 → radioconda → radioconda Prompt)를 열면 `uhd_usrp_probe`, `uhd_fft` 등의 도구와 Python 환경이 활성화됩니다.

> **드라이버 호환성 주의:** GNU Radio 번들(UHD 포함)을 쓰더라도 **USB 드라이버(WinUSB)는 3.3절대로 한 번 설치**해야 합니다. 여러 UHD가 설치되어 있어도 USB 장치 인식은 같은 WinUSB 드라이버로 공유됩니다.

### 3.7 PATH 설정 요약

- UHD 인스톨러는 보통 `C:\Program Files\UHD\bin`을 PATH에 추가합니다. 안 되어 있으면 직접 추가:

```
set PATH=C:\Program Files\UHD\bin;%PATH%
```

- GNU Radio(CASTLE 등) 인스톨러는 시작 메뉴의 **"GNU Radio Command Prompt"** 또는 배치 파일(`gnuradio-prompt.bat`)을 통해 올바른 환경을 제공합니다.

---

## 4. 하드웨어 연결 및 확인

### 4.1 데몬보드(Daughterboard) 장착

1. **전원이 꺼진 상태에서만** 데몬보드를 장착/탈착하세요. (데몬보드는 보드 양쪽 슬롯이 180도 반대 방향입니다.)
2. 데몬보드를 슬롯(A 또는 B)에 눌러 꽂습니다. 스탠드오프로 단단히 고정합니다.
3. 데몬보드가 없는 상태로 UHD를 실행하면 `uhd_usrp_probe`에서 데몬보드 ID가 "Unknown/None"으로 표시됩니다. RF 신호 테스트에는 데몬보드가 꼭 필요합니다.

### 4.2 전원 및 USB 연결

1. 전원 어댑터를 USRP1의 DC 커넥터에 연결하고 콘센트에 꽂습니다.
2. USB 케이블(B형)을 USRP1과 PC에 연결합니다.
3. PC가 부팅된 상태에서 USB를 연결하면 장치 관리자에서 장치가 잡힙니다(드라이버 설치 후).

### 4.3 LED 동작 확인

- **전원을 넣으면** 보드 LED가 켜집니다.
- **UHD가 펌웨어/FPGA를 로드하기 전까지** FPGA는 아직 프로그램이 없어 LED 상태가 불안정할 수 있습니다.
- `uhd_find_devices`나 `uhd_usrp_probe`를 실행하면 UHD가 자동으로 펌웨어와 FPGA 이미지를 로드하며, 이후 안정적으로 인식됩니다.
- 정상 로드 후에도 장치를 다시 켰을 때는 항상 UHD가 이미지를 다시 로드합니다(이미지는 RAM에만 저장되므로).

---

## 5. 장치 인식 확인

### 5.1 uhd_find_devices

```
uhd_find_devices
```

성공 시 출력 예:

```
-- UHD Find Devices --
UHD Device 0
Device Address:
    type: usrp1
    name: USRP1
    serial: 00000000
```

- 빈 결과가 나오면 [8. 문제 해결](#8-문제-해결-트러블슈팅)을 확인하세요.
- 특정 장치를 지정하려면: `uhd_find_devices --args="type=usrp1"` 또는 `--args="serial=..."`.

### 5.2 uhd_usrp_probe

```
uhd_usrp_probe
```

- `uhd_usrp_probe`는 보드 정보, FPGA 이미지, 데몬보드 ID/주파수 범위 등을 출력합니다.
- 출력 중 **"RX DSP: 0"**, **"TX DSP: 0"**, **"RX Dboard: ..."**, **"TX Dboard: ..."** 부분을 확인합니다.
- 예: WBX 데몬보드 장착 시 `RX Dboard: WBX`, `TX Dboard: WBX`가 표시되고, 주파수 범위 50MHz~2.2GHz가 나옵니다.

### 5.3 출력 예시 (요약)

```
[INFO] [UHD] Win32; Microsoft Visual C++ version 14.x; Boost_...
[INFO] [USRP1] Opening a USRP1 device...
[INFO] [USRP1] Using FPGA clock rate of 64.000000 MHz...

|   Device: USRP1 Device
|   |
|   |   /
|   |   |   Mboard: USRP1
|   |   |   |   name: USRP1
|   |   |   |   serial: 00000000
|   |   |   |   FW Version: 0.0
|   |   |   |   FPGA Version: 0.0
...
|   |   |   RX Dboard: A
|   |   |   |   ID: WBX (0x0042)
...
|   |   |   RX DSP: 0
|   |   |   |   Freq range: -32.000 to 32.000 MHz
|   |   |   |   Rates: ...
|   |   |   TX DSP: 0
|   |   |   |   Freq range: -44.000 to 44.000 MHz
...
```

> 참고: USRP1의 RX DSP 주파수 범위는 ±(master clock/2) = ±32 MHz, TX DSP는 ±44 MHz(64×0.6875)입니다.

---

## 6. 예제 실행 (USRP1에 맞는 파라미터)

> 아래 모든 명령은 UHD bin 폴더(GNU Radio 프롬프트/radioconda 프롬프트 포함)에서 실행합니다.
>
> **USRP1 권장 샘플레이트:** 16-bit(sc16) 기준 **최대 8 MS/s**. 예제에서는 2 MS/s, 4 MS/s, 8 MS/s를 사용합니다.
>
> **주파수/이득은 장착된 데몬보드 범위에 맞게** `uhd_usrp_probe`로 확인한 값을 사용하세요. (아래 예제는 WBX(50~2200 MHz) 가정, 자주 쓰는 값 100 MHz~1 GHz)

### 6.1 수신: rx_samples_to_file

RF 신호를 받아 IQ 샘플을 파일로 저장합니다.

```
rx_samples_to_file --freq 100e6 --rate 2e6 --gain 20 --duration 10 --file rx_100mhz.dat
```

- `--freq 100e6`: 수신 중심 주파수 (100 MHz, WBX 기준)
- `--rate 2e6`: 2 MS/s (USRP1 안전 범위)
- `--gain 20`: 데몬보드+코덱 이득 20 dB
- `--duration 10`: 10초 동안 수신
- `--file ...`: 저장 파일명 (기본 `usrp_samples.dat`, sc16 interleaved IQ)
- `--type short` (기본): 16-bit 정수 형식. 나중에 Python 등으로 분석하려면 `--type float`도 가능.

생성된 파일은 파이썬으로 읽을 수 있습니다:

```python
import numpy as np
iq = np.fromfile("rx_100mhz.dat", dtype=np.int16)
iq = iq[0::2] + 1j * iq[1::2]
iq = iq.astype(np.complex64) / 32768.0
print(iq.shape, iq[:5])
```

### 6.2 송신: tx_samples_from_file / tx_waveforms

**(a) 파일 재생 송신** — 위 6.1로 기록한 파일을 다시 보냅니다:

```
tx_samples_from_file --freq 100e6 --rate 2e6 --gain 10 --file rx_100mhz.dat --type short --repeat
```

- `--repeat`: 파일을 반복 송신.

**(b) 파형 생성 송신** — 직접 사인파를 만들어 보냅니다:

```
tx_waveforms --freq 100e6 --rate 2e6 --gain 10 --wave-type SINE --wave-freq 250e3 --ampl 0.5
```

- 250 kHz 사인파를 100 MHz에 실어 보냅니다.
- 스펙트럼 분석기(또는 6.4의 수신 툴)로 확인하면 100 MHz ± 0.25 MHz에 신호가 보입니다.

> ⚠️ **안전 주의:** 송신 전 안테나가 연결되어 있는지, 그리고 수신기로 직접(감쇠 없이) 신호를 보내지 않는지 확인하세요. USRP1 TX 출력은 데몬보드에 따라 수십 dBm 수준일 수 있으며, 수신 장치를 손상시킬 수 있습니다. 테스트 시 **30~40 dB 감쇠기**를 넣는 것을 권장합니다.

### 6.3 스펙트럼 확인: rx_ascii_art_dft

터미널에서 ASCII 스펙트럼으로 즉시 확인:

```
rx_ascii_art_dft --freq 100e6 --rate 2e6 --gain 20
```

FM 방송(88~108 MHz), 무선 마이크, 각종 신호원을 들이대면 스펙트럼에 피크가 보입니다.

### 6.4 성능 테스트: benchmark_rate

USB2.0 링크가 실제로 어느 속도까지 버티는지 확인합니다:

```
benchmark_rate --rate 8e6 --rxrate 4e6 --txrate 4e6 --duration 5
```

- USRP1은 sc16 기준 TX+RX 합계 8 MS/s(32 MB/s) 이내로 맞추세요.
- 예: `--rxrate 4e6` + `--txrate 4e6` = 8 MS/s.
- 오버플로(Overflow)/언더플로(Underflow)가 뜨지 않는지 확인합니다.

### 6.5 GNU Radio 예제: uhd_fft / uhd_rx_cfile / uhd_siggen

GNU Radio를 설치했다면 편리한 Python GUI 도구를 사용할 수 있습니다.

**스펙트럼 분석기:**

```
uhd_fft --freq 100e6 --rate 2e6 --gain 20
```

- FFT / Waterfall / Scope 표시를 전환할 수 있습니다. 실시간 주파수/이득/샘플레이트 변경도 가능합니다.

**IQ 파일 녹음:**

```
uhd_rx_cfile -f 100e6 -r 2e6 -g 20 -N 2000000 out.dat
```

**신호 발생기 (송신 테스트):**

```
uhd_siggen_gui --freq 100e6 --rate 2e6 --gain 10
```

- 사인/스윕/잡음 등 파형을 GUI로 생성해 송신할 수 있습니다.

> 이 도구들은 GNU Radio의 `gr-uhd`를 사용하므로, USRP1과 데몬보드가 `uhd_usrp_probe`에서 정상 인식되어야 합니다.

### 6.6 GNU Radio Companion (GRC) 플로우그래프

GRC를 열고 아래처럼 블록을 배치하면 됩니다.

**예제 1: 간단한 RX + GUI 스펙트럼**

| 블록 | 설정 |
| --- | --- |
| `UHD: USRP Source` | Samp Rate: `2e6`, Ch0: Center Freq: `100e6`, Ch0: Gain Value: `20`, Ch0: Antenna: (데몬보드별, 예: `TX/RX` 또는 `RX2`) |
| `QT GUI Frequency Sink` | FFT Size: `1024`, Center Frequency: `0`, Bandwidth: `2e6` |
| 연결 | USRP Source → Frequency Sink |

**예제 2: 간단한 TX**

| 블록 | 설정 |
| --- | --- |
| `Signal Source` | Waveform: Sine, Frequency: `250e3`, Amplitude: `0.5` |
| `UHD: USRP Sink` | Samp Rate: `2e6`, Ch0: Center Freq: `100e6`, Ch0: Gain Value: `10` |
| 연결 | Signal Source → USRP Sink |

- **Subdevice(Subdev Spec)** 를 지정해야 할 경우 데몬보드가 A 슬롯이면 `A:0`, B 슬롯이면 `B:0`으로 설정합니다.
- USRP Source/Sink 블록은 실행 시 UHD 이미지 로드 및 데몬보드 설정을 자동 수행합니다.

### 6.7 Python UHD API 예제

UHD Python wheel을 설치했다면(3.5절) 아래 스크립트로 수신할 수 있습니다.

**RX 예제 (`usrp1_rx.py`):**

```python
import numpy as np
import uhd

# USRP1 연결 (첫 번째 발견 장치)
usrp = uhd.usrp.MultiUSRP("")

# 설정
usrp.set_rx_freq(100e6, 0)     # 중심 주파수 100 MHz
usrp.set_rx_gain(20, 0)        # 이득 20 dB
usrp.set_rx_rate(2e6, 0)       # 2 MS/s

print("Device:", usrp.get_pp_string())

# 10초(2 MS/s * 10 = 2000만 샘플) 수신
samples = usrp.recv_num_samps(20_000_000, 100e6, 2e6, [0], 20)
np.save("usrp1_rx.npy", samples)

# 스펙트럼 대략 확인
import numpy as np
spec = np.abs(np.fft.fftshift(np.fft.fft(samples[::100])))  # 일부만 FFT
print("max |FFT| index:", np.argmax(spec))
```

**TX 예제 (`usrp1_tx.py`):**

```python
import numpy as np
import uhd

usrp = uhd.usrp.MultiUSRP("")
usrp.set_tx_freq(100e6, 0)
usrp.set_tx_gain(10, 0)
usrp.set_tx_rate(2e6, 0)

t = np.arange(0, 0.1, 1.0 / 2e6)               # 0.1초
wave = 0.5 * np.exp(1j * 2 * np.pi * 250e3 * t) # 250 kHz 사인파
usrp.send(wave, 2e6, 0, timeout=1.0)           # stream
print("TX done:", wave.shape)
```

### 6.8 데몬보드별 추천 주파수/이득 예시

| 데몬보드 | 주파수 범위 | 예제 주파수 | 비고 |
| --- | --- | --- | --- |
| BasicRX / BasicTX | DC ~ 250 MHz (IF) | 1~250 MHz | RF 프론트엔드 없음(IF 직접) |
| LFRX / LFTX | DC ~ 30 MHz | 5~20 MHz | 저주파 직접 |
| TVRX2 | 50 MHz ~ 860 MHz (RX only) | 100 MHz (FM), 700 MHz (DTV) | RX 전용 |
| WBX | 50 MHz ~ 2.2 GHz | 100 MHz, 1 GHz, 2.1 GHz | 가장 무난한 표준 |
| SBX | 400 MHz ~ 4.4 GHz | 1~3 GHz | |
| XCVR2450 | 2.4~2.5 / 4.9~5.9 GHz | 2.45 GHz | WiFi/WiBro 대역 |
| DBSRX | 800 MHz ~ 2.4 GHz (RX only) | 1~2 GHz | RX 전용 |

데몬보드 없이 마더보드만 있어도 `uhd_usrp_probe`는 동작하지만, 실제 RF 샘플은 데몬보드에서 나오므로 **RF 테스트 전 데몬보드 장착을 확인**하세요.

---

## 7. 파라미터 해설 (USRP1 기준)

### 7.1 샘플레이트 계산

- USRP1의 master clock은 **64 MHz**로 고정입니다.
- RX 샘플레이트 = `64 MHz ÷ decimation` (예: decim 8 → 8 MS/s, decim 32 → 2 MS/s)
- TX 샘플레이트 = `64 MHz ÷ interpolation`
- UHD가 요청한 rate에 가장 가까운 정수 decimation으로 **자동 보정(coerce)** 합니다. 예를 들어 3 MS/s를 요청하면 실제로는 64/21 ≈ 3.0476 MS/s로 동작합니다.
- **주의:** USRP1의 실효 샘플레이트는 요청값과 다를 수 있으므로, 반드시 `get_samp_rate()`로 **실제 rate를 확인**하고 신호 처리에 사용하세요.

### 7.2 Subdevice / 안테나

- 슬롯 A 데몬보드의 RX: `A:0`, 슬롯 B: `B:0` (듀얼 채널 데몬보드는 `A:0`, `A:1` 등)
- 안테나 선택 문자열은 데몬보드마다 다릅니다: 예) WBX는 `TX/RX`, TVRX2는 `RX2`, LFRX는 `RXA`/`RXB` 등. `uhd_usrp_probe` 출력의 `Antennas:` 목록을 참고하세요.

### 7.3 OTW 포맷 (sc16 / sc8)

- 기본 `sc16`: I/Q 각각 16-bit 정수, 최대 8 MS/s.
- `sc8`: I/Q 각각 8-bit 정수, **RX 전용** 최대 16 MS/s. (여러 RX 채널을 쓸 때 유용)
- 예: `rx_samples_to_file --otw sc8 --rate 12e6 ...`

### 7.4 기타

- `--gain`: 데몬보드 이득 + AD9862 PGA 이득. WBX 기준 약 0~30 dB, 사용 범위는 `uhd_usrp_probe`에 표시됩니다.
- `--mcr`: 외부 클록 개조를 하지 않았다면 64e6 고정이므로 지정할 필요 없음.
- `--ref`: USRP1은 내부 클록만 지원(`internal`). 외부 레퍼런스는 하드웨어 개조(9.3절)가 필요합니다.

---

## 8. 문제 해결 (트러블슈팅)

| 증상 | 원인 / 해결 |
| --- | --- |
| 장치 관리자에서 "알 수 없는 장치"로 표시 | WinUSB 드라이버 미설치. [3.3절](#33-usb-드라이버winusb-설치) 재실행 |
| `uhd_find_devices` 결과 없음 | ① 드라이버 확인 ② USB 포트 변경(USB2/EHCI 우선) ③ 케이블 교체 ④ 전원 연결 확인 ⑤ 재부팅 |
| `Could not locate USRP1 firmware. ... uhd_images_downloader.py` | 이미지 미다운로드. [3.4절](#34-이미지펌웨어--fpga-내려받기) 실행 후 재시도 |
| `RuntimeError: No device found` | 데몬보드/전원 문제 또는 이미지 로드 실패. 재부팅 후 `uhd_usrp_probe` 재실행 |
| Windows 드라이버 설치 마지막에 오류 문구 | 알려진 이슈. USRP 전원을 껐다 켜면 정상 동작 |
| `USB transfer failed` / `-71 (EPROTO)` | USB 신호 품질 문제. 짧은 케이블, USB 허브, 다른 포트 사용 |
| 송수신 시 Overflow / Underflow 발생 | 샘플레이트가 너무 높음. TX+RX 합계 8 MS/s 이하로 낮춤. `--spb` 값 증가 |
| `rx_samples_to_file` 실행 시 즉시 실패 | 4RX FPGA 이미지(`usrp1_fpga_4rx.rbf`) 사용 시 일부 예제가 실패. 기본 이미지로 복원(`uhd_images_downloader` 재실행 후 `--args="fpga=usrp1_fpga.rbf"`) |
| GNU Radio 블록 실행 시 `ImportError: DLL load failed` | GNU Radio 번들의 UHD와 PATH의 다른 UHD DLL 충돌. GNU Radio 전용 프롬프트에서 실행하고 PATH를 분리 |
| `uhd_usrp_probe`에서 데몬보드가 None으로 표시 | 데몬보드 미장착 또는 EEPROM 미인식. 보드 재장착, 재부팅 |

---

## 9. USRP1의 제약 사항과 고급 팁

### 9.1 지원 / 에뮬레이션되는 기능

USRP1의 FPGA는 최신 USRP(RFNoC 등)에 비해 작아서 일부 고급 기능이 **소프트웨어로 에뮬레이션**됩니다.

- **에뮬레이션되는 기능** (호스트 시계 기반이라 정밀도가 제한됨):
  - 시간 설정/조회 (`set_time_now`, `get_time_now`)
  - 특정 시간에 송수신, 정해진 개수만큼 송수신
  - TX/RX 버스트 종료 플래그
  - late command / late packet / underflow / overflow 알림
- **지원되지 않는 기능:**
  - TX/RX 버스트 시작 플래그 (`has_time_sync`, 타임 스탬프 기반 멀티채널 정밀 동기는 기대 어려움)

따라서 USRP1에서는 **실시간(비동기) 스트리밍** 위주로 사용하는 것이 바람직합니다.

### 9.2 4RX FPGA 이미지

- 기본 `usrp1_fpga.rbf`는 RX DDC 2개 + TX DUC 2개.
- `usrp1_fpga_4rx.rbf`는 RX DDC 4개 + TX 없음(RX 전용 4채널 수신, TVRX2 등 활용).

```
uhd_usrp_probe --args="fpga=usrp1_fpga_4rx.rbf"
```

> 주의: 4RX 이미지에서는 TX 예제와 일부 `rx_samples_to_file` 예제가 동작하지 않습니다.

### 9.3 외부 클록(Reference) 개조 (고급)

USRP1은 기본적으로 내부 64 MHz 클록만 사용하지만, 개조를 통해 외부 레퍼런스를 넣을 수 있습니다.

- SMA 커넥터(`LTI-SASF54GT`)를 `J2001`에 납땜
- 저항 `R2029` → `R2030`, 커패시터 `C925` → `C926` 이동, `C924` 제거
- 외부 클록: 7~15 dBm 정방파
- 이후 `usrp_burn_mb_eeprom --values="mcr=<rate>"`로 EEPROM에 기록

일반 사용자는 권장하지 않으며, MIMO/위상 동기화가 꼭 필요한 경우에만 시도하세요.

### 9.4 MIMO 사용

- USRP1은 슬롯 2개 + 코덱 2개로 **2×2 MIMO**가 가능하지만, USB 2.0 대역폭 때문에 **전 채널 합산 8 MS/s 이하**로 제한됩니다.
- 두 슬롯에 같은 데몬보드를 장착하면 `--channels 0,1`(또는 `A:0 B:0`)로 두 채널 동시 수신이 가능합니다. 채널 수가 늘면 각 채널 샘플레이트를 낮춰야 합니다.

### 9.5 기타 팁

- USRP1은 RAM 기반 로딩이므로 **매번 PC에서 이미지를 로드**합니다. 처음 실행 시 수 초가 걸릴 수 있습니다.
- 오래된 보드라면 **접점/커넥터 청소**, 데몬보드 EEPROM 배터리(리튬 셀) 점검을 권장합니다.
- 가능하면 UHD 예제 폴더(`C:\Program Files\UHD\lib\uhd\examples\python`)의 Python 예제(`rx_to_file.py`, `rx_spectrum_to_pyplot.py` 등)도 참고하세요.

---

## 10. 참고 자료

| 자료 | 링크 |
| --- | --- |
| USRP1 KB 페이지 (Ettus) | https://kb.ettus.com/USRP1 |
| UHD 메뉴얼 – USRP1 | https://files.ettus.com/manual/page_usrp1.html |
| UHD 메뉴얼 – Windows 설치 | https://files.ettus.com/manual/page_install_binary.html |
| UHD 메뉴얼 – 이미지 다운로더 | https://files.ettus.com/manual/page_images.html |
| UHD 최신 Windows 인스톨러 | http://files.ettus.com/binaries/uhd/latest_release |
| UHD Windows USB 드라이버 | http://files.ettus.com/binaries/misc/erllc_uhd_winusb_driver.zip |
| GNU Radio Windows 설치 안내 | https://wiki.gnuradio.org/index.php/WindowsInstall |
| RadioConda (권장 GNU Radio 설치) | https://github.com/ryanvolz/radioconda |
| CASTLE GNU Radio for Windows | https://www.castle.cloud/gnuradio-for-windows/download/ |
| GNU Radio 공식 문서 | https://www.gnuradio.org |
| UHD 예제 목록 (공식) | https://github.com/EttusResearch/uhd/tree/master/host/examples |
| UHD 소스 (USRP1 구현) | https://github.com/EttusResearch/uhd/tree/master/host/lib/usrp/usrp1 |

---

*문서 생성일: 2026-08-02 · Windows 10/11 + UHD 4.9 기준*
