# 樹莓派 Zero 2 W — mesh 網路下 Wi-Fi `ASSOC-REJECT status_code=16`:根因與修法

**語言：** [English](README.md) · **繁體中文** · [日本語](README.ja.md) · [한국어](README.ko.md)

一份實戰筆記:為什麼 **Raspberry Pi Zero 2 W**(Broadcom **BCM43430** / `brcmfmac` 驅動)在 **Wi-Fi mesh** 路由器上反覆關聯失敗 —— `CTRL-EVENT-ASSOC-REJECT ... status_code=16` —— 以及真正修好它的方法。含逐變數隔離的受控實驗、三台裝置對照,並清楚標出哪些結論有上游 issue 背書、哪些是親手實測。

> **太長不看(TL;DR)** —— 在老的 BCM43430 晶片上:
> 1. **用純 WPA2-PSK。** 純 WPA3/SAE 在這顆晶片上**必定失敗**;WPA2/WPA3 **混合模式**連得上但**時好時壞**(訊號強冷開機 OK,訊號弱/熱重開不可靠)。
> 2. **在 `/etc/modprobe.d/brcmfmac.conf` 寫 `options brcmfmac roamoff=1`** —— 關掉韌體漫遊引擎,而那正是 **mesh band steering** 會卡住的東西。本網實測**單靠它就 ~88% 冷開機自動連強節點 —— 不用鎖 BSSID、可攜**。
> 3. **(選擇性)鎖一台強 AP 的 BSSID** —— 只在「要保證每次都連強 AP」時才用,它有效但脆(換路由器就壞)。
>
> **Wi-Fi 6 不是元兇**(常見誤判 —— 下方有證)。**別盲目降韌體** —— 網路上熱門的「降到 9.88.4.65」是針對**另一個晶片版本**,對 BCM43430/1 不適用。

---

## 硬體說明(stock Zero 2 W 也適用嗎?)

**適用 —— 而且若你是原廠機反而更重要。** 原廠 Pi Zero 2 W **只有 Wi-Fi**、沒有乙太網路,所以 Wi-Fi 是你**唯一**的連線,下面的修法非做不可。

本筆記的機器額外裝了 **USB 乙太 HAT**(Waveshare ETH/USB HUB,Realtek **RTL8152** 10/100),才多了一個有線 `eth0`,因此能跑「**有線為主、Wi-Fi 備援**」——這是最穩的架構,但**多數讀者沒有**。若你也加這種 HAT,請留意下方警告:hub 由 Pi 供 5V,供電瞬斷會讓整條 USB 匯流排(含乙太)掉線。

## 症狀

```
wpa_supplicant: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... auth_failures=N reason=CONN_FAILED
```

- `nmcli con up <SSID>` 反覆失敗,最後報 `Error: ... Secrets were required, but not provided` —— 這是 NetworkManager 在多次關聯失敗後**誤判**成密碼問題。**密碼其實是對的。**
- `bssid=00:00:00:00:00:00`(全零)代表**韌體在 4-way 握手前就擋掉關聯** → 是 AP 加密/模式不相容,不是帳密問題。

## 根因

Zero 2 W 的 Wi-Fi 晶片是老的 **BCM43430**(韌體 `7.45.96.s1`,2023)。在 mesh 的 `*_2.4G` SSID 上,有兩件事重要:

1. **加密。** 純 WPA3/SAE **必定**讓老韌體拒絕關聯。WPA2/WPA3 **混合模式**其實**連得上**(客戶端會協商 PSK over SAE),但**不可靠** —— 訊號強時 ~1–2 秒連上,訊號弱或熱重開時要被擋很多次(實測拖到約 35 秒)甚至失敗。**只有純 WPA2-PSK 才完全可靠。**
2. **band steering 才是主兇**:mesh 在關聯途中把客戶端導去較弱的節點然後 reject(`status_code=16`;特徵:reject 那行的 `bssid=` 跟你要連的節點不同,或全零)。解法是 **`roamoff=1`**(關韌體漫遊引擎),本網單靠它就夠;**鎖 BSSID 是選擇性的** —— 它只是*保證*特定 AP,代價是可攜性。

## 修法

**路由器端** —— 把 2.4GHz SSID 設成**純 WPA2-Personal**(關掉 WPA3/混合)。**Wi-Fi 6 可以保持開啟**(實測無害 —— 別為一台老客戶端拖慢整個網路)。

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

BSSID 就是**某一台 AP 無線電的 MAC 位址**。鎖它只是針對 **mesh band steering** 的 workaround,而且很脆:

- **換路由器 / mesh 就會失效。** 新 AP 是新的 MAC,你鎖的 BSSID 不存在了 → Pi 永遠連不上,直到你更新它。
- 被鎖的節點若斷電/搬走/壞掉,Pi **不會漫遊**去別的節點(你既鎖了 BSSID 又設了 `roamoff=1`)。

換硬體時,重新掃描更新即可,或乾脆解除鎖定:

```bash
# 更新成新的強節點
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid <NEW_BSSID>
# 或完全解除鎖定(若新路由器是單一 AP、非 mesh,就這樣做)
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid ""
```

**不是每種環境都要鎖。** 它只為 band-steering 的 mesh 而設。若是**單一 AP(非 mesh)**,純 WPA2 + `roamoff=1` 通常就夠 —— 不用鎖 BSSID、可攜性最好。最乾淨的解法是 **AP 端**:給老客戶端開一個關掉 band steering 的專用 SSID,客戶端就完全不必鎖任何東西。

## 延伸閱讀

- 📄 **[完整排錯 writeup(繁中)](docs/troubleshooting.zh-TW.md)** —— 受控實驗、三台對照、韌體版本分析、「哪些有效/哪些沒用」、完整來源清單。
- 📄 其他語言詳細版:[English](docs/troubleshooting.en.md) · [日本語](docs/troubleshooting.ja.md) · [한국어](docs/troubleshooting.ko.md)

## ⚠️ 血淚警告

**別對這顆晶片做密集背靠背重連或重開機迴圈壓測。** 在 Zero 2 W 上,密集 Wi-Fi 重新關聯會讓 BCM43430 韌體 halt、進而拖垮**整塊板子**(連 USB 乙太也死,只能斷電重來)。重連間隔 **≥10 秒就很穩**(實測 100/100)。這個脆弱性是 Zero 2 W 老晶片專屬 —— Pi 3B+/Pi 4(CYW43455 / BCM4345-6)同樣測試撐過 1000/1000。

## 致謝

來自真實家用部署的現場筆記(三台樹莓派接在小米 Wi-Fi 6 mesh 上)。所有主機相關機密(Wi-Fi 密碼、SSID、BSSID、內網 IP)都已換成佔位符。

## 授權

[MIT](LICENSE)
