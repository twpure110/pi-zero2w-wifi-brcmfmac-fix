# 라즈베리 파이 Zero 2 W — 메시 네트워크에서 Wi-Fi `ASSOC-REJECT status_code=16`: 원인과 해결

**언어:** [English](README.md) · [繁體中文](README.zh-TW.md) · [日本語](README.ja.md) · **한국어**

**Raspberry Pi Zero 2 W**(Broadcom **BCM43430** / `brcmfmac` 드라이버)가 **Wi-Fi 메시** 공유기에 연결(association)을 반복적으로 실패하는 문제 —— `CTRL-EVENT-ASSOC-REJECT ... status_code=16` —— 의 근본 원인과 실제로 해결한 방법에 대한 현장 기록입니다. 변수를 하나씩 분리한 대조 실험, 3대 비교, 그리고 어떤 결론이 업스트림 이슈로 뒷받침되고 어떤 것이 직접 측정한 것인지를 명확히 구분합니다.

> **요약(TL;DR)** —— 오래된 BCM43430 칩에서:
> 1. **순수 WPA2-PSK 사용.** 순수 WPA3/SAE는 이 칩에서 **항상 실패**; WPA2/WPA3 **혼합 모드**는 연결은 되지만 불안정(강신호 콜드 부팅은 OK, 약신호/웜 재부팅은 불안정). **⚠️ `wpasupplicant 2:2.10-24+rpt3`(2026-11)부터는 혼합 모드가 아예 연결되지 않음 —— 아래 [2026-11 업데이트](#2026-11-update) 참고.**
> 2. **`/etc/modprobe.d/brcmfmac.conf` 에 `options brcmfmac roamoff=1`** —— 펌웨어 로밍 엔진을 끄며, 바로 그것이 **메시 밴드 스티어링**이 걸리는 지점. 우리 네트워크에서는 이것만으로 **콜드 부팅의 ~88%에서 강한 노드에 자동 연결 —— BSSID 고정 불필요, 이식성 유지**.
> 3. **(선택) 강한 AP의 BSSID 고정**은 매번 강한 AP를 *보장*해야 할 때만 —— 작동하지만 취약함(공유기를 바꾸면 깨짐).
>
> **Wi-Fi 세대(6/7)는 범인이 아닙니다**(흔한 오진 —— 아래에서 증명). **무턱대고 펌웨어를 다운그레이드하지 마세요** —— 유명한 "9.88.4.65로 다운그레이드"는 **다른 칩 리비전**용이며 BCM43430/1에는 적용되지 않습니다.

<a id="2026-11-update"></a>
## ⚠️ 2026-11 업데이트: `wpasupplicant 2:2.10-24+rpt3` 이후 WPA2/WPA3 혼합 모드 연결 불가

Raspberry Pi OS(trixie)에서 일상적인 `apt full-upgrade` 를 실행했더니 `wpasupplicant` 가 `2:2.10-24` 에서 **`2:2.10-24+rpt3`** 으로 올라갔고 —— Zero 2 W가 **혼합 모드 SSID에 전혀 연결되지 않게 되었습니다**(몇 달 동안 문제없이 연결되던 같은 네트워크):

```
wpa_supplicant: wlan0: Authentication with <BSSID> timed out.
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... reason=CONN_FAILED
NetworkManager: device (wlan0): Activation: (wifi) association took too long
NetworkManager: device (wlan0): state change: config -> failed (reason 'no-secrets')
```

그 이후 NetworkManager는 **자동 재시도를 멈춥니다**(`autoconnect-retries 0` 이어도 마찬가지). 그래서 수동으로 다시 연결하기 전까지 Wi-Fi는 끊긴 상태로 남습니다.

- **유력한 원인:** `+rpt3` changelog에 *"Add upstream patch to fix reporting of SAE support over D-Bus"* 라고 되어 있습니다. NetworkManager가 supplicant의 SAE 지원을 인식하게 되어 혼합 모드 AP에 SAE(WPA3)로 연결하려 하는데 —— 이 칩은 SAE를 못 합니다. 이전에는 PSK로 폴백했습니다. (changelog에서 추론한 것이며, 실제로 확인한 것은 "다운그레이드만으로 즉시 해결됨"입니다.)
- **해결 A(권장): 공유기를 순수 WPA2-PSK로.** TL;DR과 동일. 고정할 것이 없고 이후 업그레이드도 안전합니다.
- **해결 B: 다운그레이드 후 hold**(혼합 모드를 꼭 유지해야 한다면):
  ```bash
  sudo apt-get install --allow-downgrades wpasupplicant=2:2.10-24
  sudo apt-mark hold wpasupplicant
  sudo nmcli con up "<YOUR_SSID_2G>"   # 'no-secrets' 이후 NM은 스스로 재시도하지 않음
  ```
  결과: 첫 시도에 연결, 거부 0회; 새 커널로 재부팅한 뒤에도 자동 연결됨. 대가: hold 동안 `wpasupplicant` 는 보안 업데이트를 받지 못합니다 —— 공유기를 순수 WPA2로 바꾸면(또는 수정된 패키지가 나오면) `sudo apt-mark unhold wpasupplicant`.
- **영향 없음:** 같은 업그레이드로 `firmware-brcm80211` 도 20260519로 올라갔지만, 이 보드가 실제로 로드하는 파일(`brcmfmac43436s-sdio.bin`)은 여전히 `7.45.96.s1` —— Wi-Fi 펌웨어는 바뀌지 않았습니다. 본인 환경은 `journalctl -k -b | grep 'Firmware: BCM'` 으로 확인하세요.

---

## 하드웨어 참고(기본 Zero 2 W에도 해당되나요?)

**해당됩니다 —— 기본 보드라면 오히려 더 중요합니다.** 기본 Pi Zero 2 W는 **Wi-Fi 전용**이라 이더넷이 없으므로 Wi-Fi가 *유일한* 연결이고, 아래 해결책은 필수입니다.

이 글의 보드에는 **USB 이더넷 HAT**(Waveshare ETH/USB HUB, Realtek **RTL8152** 10/100)를 추가해 유선 `eth0`도 있습니다. 덕분에 "**유선 우선, Wi-Fi 예비**"라는 가장 견고한 구성이 가능하지만, **대부분의 독자에게는 없습니다**. 이런 HAT를 추가한다면 아래 경고에 유의하세요: 허브가 Pi에서 5V를 끌어오므로, 전압 강하 시 USB 버스 전체(이더넷 포함)가 끊길 수 있습니다.

## 증상

```
wpa_supplicant: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... auth_failures=N reason=CONN_FAILED
```

- `nmcli con up <SSID>` 가 반복 실패하고 결국 `Error: ... Secrets were required, but not provided` 라고 나옵니다 —— 이는 여러 번의 association 실패 후 NetworkManager가 비밀번호 문제로 **오인**한 것입니다. **비밀번호는 맞습니다.**
- `bssid=00:00:00:00:00:00`(전부 0)은 **4-way 핸드셰이크 이전**에 펌웨어가 association을 거부했다는 뜻 → AP 암호화/모드 비호환이며 자격 증명 문제가 아닙니다.

## 근본 원인

Zero 2 W의 Wi-Fi 칩은 오래된 **BCM43430**(펌웨어 `7.45.96.s1`, 2023)입니다. 메시의 `*_2.4G` SSID에서 두 가지가 중요합니다:

1. **암호화.** 순수 WPA3/SAE는 **항상** 오래된 펌웨어가 association을 거부하게 만듭니다. WPA2/WPA3 **혼합 모드**는 **연결 가능**(클라이언트가 SAE 대신 PSK를 협상)하지만 **불안정** —— 강신호에서는 ~1–2초에 연결, 약신호나 웜 재부팅에서는 여러 번 거부(실측 약 35초)되거나 실패. **완전히 신뢰할 수 있는 것은 순수 WPA2-PSK뿐입니다.** (혼합 모드가 연결되는 것은 `wpasupplicant` ≤ `2:2.10-24` 일 때뿐 —— [2026-11 업데이트](#2026-11-update) 참고.)
2. **밴드 스티어링이 진짜 주범**: 메시가 association 도중 더 약한 노드로 넘기고 거부합니다(`status_code=16`; 특징 —— 거부 줄의 `bssid=` 가 연결하려던 노드와 다름, 또는 전부 0). 해결책은 **`roamoff=1`**(펌웨어 로밍 엔진 끄기)이고, 우리 네트워크에서는 이것만으로 충분; **BSSID 고정은 선택** —— 특정 AP를 *보장*할 뿐이며 대가는 이식성.

## 해결

**공유기 측** —— 2.4GHz SSID를 **순수 WPA2-Personal** 로 설정(WPA3/혼합 끄기). **Wi-Fi 6/7은 켜둔 채로 OK**(무해함이 검증됨 —— 오래된 클라이언트 하나 때문에 전체 네트워크를 느리게 하지 말 것. 이 테스트 네트워크는 실제로는 Wi-Fi 7; 802.11n 클라이언트는 어차피 AP의 HE/EHT를 무시함).

**클라이언트 측** —— 순수 WPA2 + 펌웨어 로밍 끄기(`roamoff=1`); BSSID 고정은 선택:

```bash
# 1) WPA2-PSK 강제 + 부팅 시 무한 재시도(BSSID 고정 없음 —— roamoff가 스티어링 처리)
sudo nmcli con modify "<YOUR_SSID_2G>" \
  802-11-wireless-security.key-mgmt wpa-psk \
  802-11-wireless-security.psk '<YOUR_WIFI_PASSWORD>' \
  802-11-wireless-security.pmf disable \
  connection.autoconnect yes connection.autoconnect-retries 0

# 2) 드라이버 계층: 펌웨어 로밍 끄기 —— 이것이 밴드 스티어링을 이기는 핵심
echo 'options brcmfmac roamoff=1' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo modprobe -r brcmfmac && sudo modprobe brcmfmac   # 또는 재부팅

# 3) (선택) 강한 AP 고정 —— roamoff만으로 약한 노드에 계속 걸릴 때만
#    찾기: sudo iw dev wlan0 scan | grep -iB5 'SSID: <YOUR_SSID_2G>' | grep -iE 'BSS|signal'
# sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid AA:BB:CC:DD:EE:FF
```

암호화가 정말 WPA2 전용(SAE 없음)인지 확인: `sudo iw dev wlan0 scan | grep -iA22 'SSID: <YOUR_SSID_2G>' | grep Authentication` 가 `Authentication suites: PSK` 여야 하며 —— `PSK SAE` 면 **안 됩니다**.

### ⚠️ 이식성: 고정된 BSSID는 AP 하드웨어에 묶입니다

BSSID는 **특정 AP 무선의 MAC 주소**입니다. 고정은 어디까지나 **메시 밴드 스티어링 전용** 우회책이며 취약합니다:

- **공유기/메시를 바꾸면 동작이 멈춥니다.** 새 AP는 다른 MAC이므로 고정한 BSSID가 존재하지 않고, 갱신하기 전까지 Pi는 절대 연결되지 않습니다.
- 고정한 노드가 꺼지거나 이동/고장하면 Pi는 다른 노드로 **로밍하지 않습니다**(BSSID 고정 + `roamoff=1`).

하드웨어를 바꾸면 다시 스캔해 갱신하거나, 고정을 해제하면 됩니다:

```bash
# 새 강한 노드로 갱신
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid <NEW_BSSID>
# 또는 고정을 완전히 해제(새 공유기가 단일 AP・비메시라면 이렇게)
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid ""
```

**항상 고정이 필요한 것은 아닙니다.** 밴드 스티어링이 있는 메시 전용입니다. **단일 AP(비메시)**라면 순수 WPA2 + `roamoff=1` 로 보통 충분 —— BSSID 고정 없이 이식성도 최상. 가장 깔끔한 해법은 **AP 측**: 오래된 클라이언트용으로 밴드 스티어링을 끈 전용 SSID를 만들면 클라이언트는 아무것도 고정할 필요가 없습니다.

## 더 읽기

- 📄 **[전체 문제 해결 기록(한국어)](docs/troubleshooting.ko.md)** —— 대조 실험, 3대 비교, 펌웨어 버전 분석, "효과 있었던 것/없었던 것", 전체 출처 목록.
- 📄 다른 언어 상세판: [English](docs/troubleshooting.en.md) · [繁體中文](docs/troubleshooting.zh-TW.md) · [日本語](docs/troubleshooting.ja.md)

## ⚠️ 어렵게 얻은 경고

**이 칩을 빠른 연속 재연결이나 재부팅 루프로 스트레스 테스트하지 마세요.** Zero 2 W에서는 조밀한 Wi-Fi 재association이 BCM43430 펌웨어를 정지(halt)시키고 **보드 전체**를 함께 멈추게 합니다(USB 이더넷도 죽고, 전원을 껐다 켜야만 복구). 재연결 간격이 **10초 이상이면 안정적**입니다(실측 100/100). 이 취약성은 Zero 2 W의 오래된 칩 고유 —— Pi 3B+/Pi 4(CYW43455 / BCM4345-6)는 같은 테스트를 1000/1000으로 버팁니다.

## 크레딧

실제 가정 배포(Xiaomi Wi-Fi 7 메시에 라즈베리 파이 3대) 현장 기록. 호스트 고유 비밀 정보(Wi-Fi 비밀번호, SSID, BSSID, 내부 IP)는 모두 플레이스홀더로 치환했습니다.

## 라이선스

[MIT](LICENSE)
