# 樹莓派 Zero 2 W — mesh 網路下 Wi-Fi `ASSOC-REJECT status_code=16`:根因與修法

**語言：** [English](README.md) · **繁體中文** · [日本語](README.ja.md) · [한국어](README.ko.md)

一份實戰筆記:**Raspberry Pi Zero 2 W**(Broadcom **BCM43430** / `brcmfmac` 驅動)為什麼在 **Wi-Fi mesh** 路由器上一直關聯失敗 —— `CTRL-EVENT-ASSOC-REJECT ... status_code=16` —— 還有到底怎麼把它修好。裡面有一個變數一個變數隔離的受控實驗、三台裝置對照,也清楚標出哪些結論有上游 issue 撐腰、哪些純粹是我自己實測出來的。

> **太長不看(TL;DR)** —— 在老的 BCM43430 晶片上:
> 1. **用純 WPA2-PSK。** 純 WPA3/SAE 在這顆晶片上**一定連不上**;WPA2/WPA3 **混合模式**連得上但**時好時壞**(訊號強冷開機 OK,訊號弱/熱重開就不可靠)。**⚠️ 從 `wpasupplicant 2:2.10-24+rpt3`(2026-11)起,混合模式直接完全連不上 —— 看下面的 [2026-11 更新](#2026-11-update)。**
> 2. **在 `/etc/modprobe.d/brcmfmac.conf` 寫 `options brcmfmac roamoff=1`** —— 把韌體漫遊引擎關掉,**mesh band steering** 卡的就是這個。我這個網實測,**光靠它就 ~88% 冷開機自己連上強節點 —— 不用鎖 BSSID、又可攜**。
> 3. **(可有可無)鎖一台強 AP 的 BSSID** —— 只在「非得保證每次都連強 AP」時才用,它有效但脆(換路由器就壞)。
>
> **Wi-Fi 世代(6/7)不是元兇**(常見誤判 —— 下面有證)。**別亂降韌體** —— 網路上很紅的「降到 9.88.4.65」是別顆晶片版本的事,對 BCM43430/1 不適用。

<a id="2026-11-update"></a>
## ⚠️ 2026-11 更新:`wpasupplicant 2:2.10-24+rpt3` 讓 WPA2/WPA3 混合模式整個連不上

在 Raspberry Pi OS(trixie)上跑了一次例行的 `apt full-upgrade`,`wpasupplicant` 從 `2:2.10-24` 升到 **`2:2.10-24+rpt3`** —— 結果 Zero 2 W **連混合模式的 SSID 直接完全連不上**(同一個網,之前好幾個月都連得好好的):

```
wpa_supplicant: wlan0: Authentication with <BSSID> timed out.
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... reason=CONN_FAILED
NetworkManager: device (wlan0): Activation: (wifi) association took too long
NetworkManager: device (wlan0): state change: config -> failed (reason 'no-secrets')
```

而且之後 NetworkManager **就不再自動重試**(就算設了 `autoconnect-retries 0` 也一樣),Wi-Fi 會一直斷著,直到你手動把它拉起來。

- **八成是這個原因:** `+rpt3` 的 changelog 寫 *「Add upstream patch to fix reporting of SAE support over D-Bus」*。NetworkManager 從此知道 supplicant 支援 SAE,就會對混合模式 AP 用 SAE(WPA3)去連 —— 偏偏這顆晶片做不到。以前它會退回用 PSK。(這是從 changelog 推出來的;實際驗證到的是:光把它降版,馬上就連上。)
- **解法 A(建議):路由器改純 WPA2-PSK。** 跟上面 TL;DR 一樣。什麼都不用鎖,以後升級也安全。
- **解法 B:降版並鎖住**(非留混合模式不可的話):
  ```bash
  sudo apt-get install --allow-downgrades wpasupplicant=2:2.10-24
  sudo apt-mark hold wpasupplicant
  sudo nmcli con up "<YOUR_SSID_2G>"   # NM 報 'no-secrets' 之後不會自己重試
  ```
  結果:第一次就連上、0 次 reject;重開機換上新 kernel 之後也照樣自動連上。代價:鎖住期間 `wpasupplicant` 收不到安全更新 —— 等路由器改成純 WPA2(或官方出了修好的版本),再跑 `sudo apt-mark unhold wpasupplicant`。
- **沒受影響的部分:** 同一次升級也把 `firmware-brcm80211` 升到 20260519,但這塊板子實際載入的檔案(`brcmfmac43436s-sdio.bin`)還是 `7.45.96.s1` —— Wi-Fi 韌體沒變。查你自己的:`journalctl -k -b | grep 'Firmware: BCM'`。

---

## 硬體說明(stock Zero 2 W 也適用嗎?)

**適用 —— 而且你是原廠機的話反而更重要。** 原廠 Pi Zero 2 W **只有 Wi-Fi**、沒有網路孔,所以 Wi-Fi 就是你**唯一**的連線,下面這套修法非做不可。

我這台另外裝了 **USB 乙太 HAT**(Waveshare ETH/USB HUB,Realtek **RTL8152** 10/100),才多一個有線 `eth0`,所以才玩得起「**有線當主力、Wi-Fi 當備援**」——這是最穩的架構,但**多數人沒有**。你要是也裝這種 HAT,注意下面的警告:hub 吃 Pi 的 5V,供電一抖,整條 USB 匯流排(連乙太一起)就會掉。

## 症狀

```
wpa_supplicant: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... auth_failures=N reason=CONN_FAILED
```

- `nmcli con up <SSID>` 一直失敗,最後丟 `Error: ... Secrets were required, but not provided` —— 這是 NetworkManager 關聯失敗太多次之後**誤判**成密碼問題。**密碼其實是對的。**
- `bssid=00:00:00:00:00:00`(全零)代表**韌體在 4-way 握手之前就把關聯擋掉了** → 是 AP 的加密/模式不合,不是帳密問題。

## 根因

Zero 2 W 用的是老晶片 **BCM43430**(韌體 `7.45.96.s1`,2023)。在 mesh 的 `*_2.4G` SSID 上,要顧兩件事:

1. **加密。** 純 WPA3/SAE **一定**讓老韌體拒絕關聯。WPA2/WPA3 **混合模式**其實**連得上**(客戶端會去談 PSK 不談 SAE),但**不可靠** —— 訊號強時 ~1–2 秒連上,訊號弱或熱重開時會被擋很多次(實測拖到約 35 秒)甚至連不上。**只有純 WPA2-PSK 才完全可靠。**(混合模式連得上的前提是 `wpasupplicant` ≤ `2:2.10-24` —— 看 [2026-11 更新](#2026-11-update)。)
2. **band steering 才是主兇**:mesh 在關聯到一半把客戶端導去比較弱的節點、然後 reject(`status_code=16`;特徵:reject 那行的 `bssid=` 跟你要連的節點對不上,或全零)。解法是 **`roamoff=1`**(關韌體漫遊引擎),我這個網單靠它就夠;**鎖 BSSID 可有可無** —— 它只是*保證*連某台 AP,代價是不可攜。

## 修法

**路由器端** —— 把 2.4GHz SSID 設成**純 WPA2-Personal**(關掉 WPA3/混合)。**Wi-Fi 6/7 開著就好**(實測無害 —— 別為了一台老客戶端把整個網拖慢)。

**客戶端** —— 純 WPA2 + 關掉韌體漫遊(`roamoff=1`);鎖 BSSID 選擇性:

```bash
# 1) 強制 WPA2-PSK + 開機無限重試(不鎖 BSSID —— roamoff 處理 steering)
sudo nmcli con modify "<YOUR_SSID_2G>" \
  802-11-wireless-security.key-mgmt wpa-psk \
  802-11-wireless-security.psk '<YOUR_WIFI_PASSWORD>' \
  802-11-wireless-security.pmf disable \
  connection.autoconnect yes connection.autoconnect-retries 0

# 2) 驅動層:關韌體漫遊 —— 這才是真正打贏 band steering 的步驟
echo 'options brcmfmac roamoff=1' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo modprobe -r brcmfmac && sudo modprobe brcmfmac   # 或重開機

# 3)(選擇性)鎖強 AP —— 只在 roamoff 還是一直連到弱節點時
#    找它:sudo iw dev wlan0 scan | grep -iB5 'SSID: <YOUR_SSID_2G>' | grep -iE 'BSS|signal'
# sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid AA:BB:CC:DD:EE:FF
```

確認加密真的只有 WPA2(無 SAE):`sudo iw dev wlan0 scan | grep -iA22 'SSID: <YOUR_SSID_2G>' | grep Authentication` 必須是 `Authentication suites: PSK` —— **不能**是 `PSK SAE`。

### ⚠️ 可攜性:鎖 BSSID = 綁定你那台 AP 的硬體

BSSID 就是**某一台 AP 無線電的 MAC 位址**。鎖它純粹是對付 **mesh band steering** 的 workaround,而且很脆:

- **換路由器 / mesh 就掛了。** 新 AP 是新 MAC,你鎖的那個 BSSID 不存在了 → Pi 永遠連不上,直到你去更新。
- 被鎖的節點要是斷電/搬走/壞掉,Pi **不會漫遊**去別的節點(你既鎖了 BSSID 又設了 `roamoff=1`)。

換硬體的時候,重掃一遍更新就好,或乾脆把鎖解掉:

```bash
# 更新成新的強節點
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid <NEW_BSSID>
# 或完全解除鎖定(若新路由器是單一 AP、非 mesh,就這樣做)
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid ""
```

**不是每種環境都要鎖。** 它只為有 band-steering 的 mesh 而設。如果是**單一 AP(不是 mesh)**,純 WPA2 + `roamoff=1` 通常就夠了 —— 不用鎖 BSSID,可攜性也最好。最乾淨的做法是 **AP 端**:幫老客戶端開一個關掉 band steering 的專用 SSID,客戶端什麼都不用鎖。

## 延伸閱讀

- 📄 **[完整排錯 writeup(繁中)](docs/troubleshooting.zh-TW.md)** —— 受控實驗、三台對照、韌體版本分析、「哪些有效/哪些沒用」、完整來源清單。
- 📄 其他語言詳細版:[English](docs/troubleshooting.en.md) · [日本語](docs/troubleshooting.ja.md) · [한국어](docs/troubleshooting.ko.md)

## ⚠️ 血淚警告

**別對這顆晶片做密集的背靠背重連、或重開機迴圈壓測。** 在 Zero 2 W 上,密集的 Wi-Fi 重關聯會讓 BCM43430 韌體 halt、然後把**整塊板子**拖垮(連 USB 乙太一起死,只能拔電重來)。重連間隔 **≥10 秒就很穩**(實測 100/100)。這個脆弱是 Zero 2 W 老晶片獨有的 —— Pi 3B+/Pi 4(CYW43455 / BCM4345-6)同樣的測試撐過 1000/1000。

## 致謝

來自真實家用部署的現場筆記(三台樹莓派掛在小米 **Wi-Fi 7** mesh 上)。所有跟主機有關的機密(Wi-Fi 密碼、SSID、BSSID、內網 IP)都換成佔位符了。

## 授權

[MIT](LICENSE)
