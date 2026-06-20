# Raspberry Pi Zero 2 W — メッシュ環境での Wi-Fi `ASSOC-REJECT status_code=16`:原因と対処

**言語：** [English](README.md) · [繁體中文](README.zh-TW.md) · **日本語** · [한국어](README.ko.md)

**Raspberry Pi Zero 2 W**(Broadcom **BCM43430** / `brcmfmac` ドライバ)が **Wi-Fi メッシュ**ルーターへの接続(アソシエーション)に繰り返し失敗する —— `CTRL-EVENT-ASSOC-REJECT ... status_code=16` —— その原因と、実際に解決した方法の実地メモです。変数を一つずつ切り分けた対照実験、3 台比較、そして「どの結論が上流 issue に裏付けられ、どれが自分の実測か」を明示しています。

> **要点（TL;DR）** —— 古い BCM43430 チップでは:
> 1. **純粋な WPA2-PSK を使う。** 純粋な WPA3/SAE はこのチップで**必ず失敗**;WPA2/WPA3 **混在モード**は繋がるが**不安定**（強信号のコールド起動なら OK、弱信号/ウォーム再起動では不安定）。
> 2. **`/etc/modprobe.d/brcmfmac.conf` に `options brcmfmac roamoff=1`** —— ファームのローミングエンジンを無効化する。これこそ**メッシュのバンドステアリング**が引っかかる部分。本ネットワークではこれ単独で **コールド起動の ~88% で強ノードに自動接続 —— BSSID 固定不要、可搬のまま**。
> 3. **（任意）強 AP の BSSID を固定**するのは、毎回強 AP を*保証*したい場合のみ —— 有効だがもろい（ルーターを変えると壊れる）。
>
> **Wi-Fi 6 は犯人ではありません**（よくある誤診 —— 下で証明）。**安易にファームを下げないこと** —— 有名な「9.88.4.65 へダウングレード」は**別のチップ版**向けで、BCM43430/1 には当てはまりません。

---

## ハードウェアの注意（素の Zero 2 W にも当てはまる？）

**当てはまります —— 素の基板ならむしろ重要です。** 素の Pi Zero 2 W は **Wi-Fi のみ**で有線がないため、Wi-Fi が*唯一*の経路であり、下記の対処は必須です。

この記事の基板には **USB イーサ HAT**（Waveshare ETH/USB HUB、Realtek **RTL8152** 10/100）を追加してあり、有線 `eth0` も持っています。そのため「**有線を主・Wi-Fi を予備**」という最も堅牢な構成にできますが、**多くの読者にはこれがありません**。この種の HAT を追加する場合は下記の警告に注意：hub は Pi から 5V を取るため、電圧降下で USB バス全体（イーサ含む）が落ちることがあります。

## 症状

```
wpa_supplicant: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... auth_failures=N reason=CONN_FAILED
```

- `nmcli con up <SSID>` が繰り返し失敗し、最後に `Error: ... Secrets were required, but not provided` と出る —— これは何度もアソシエーションに失敗した後、NetworkManager が**誤って**パスワード問題と報告しているだけ。**パスワードは正しい**。
- `bssid=00:00:00:00:00:00`（オールゼロ）= **4-way ハンドシェイクの前**にファームがアソシエーションを拒否 → AP の暗号化/モード非互換であり、認証情報の問題ではない。

## 原因

Zero 2 W の Wi-Fi チップは古い **BCM43430**（ファーム `7.45.96.s1`、2023）。メッシュの `*_2.4G` SSID では 2 つが重要:

1. **暗号化。** 純粋な WPA3/SAE は**必ず**古いファームにアソシエーションを拒否させる。WPA2/WPA3 **混在モード**は**繋がる**（クライアントが SAE より PSK を選ぶ）が**不安定** —— 強信号なら ~1–2 秒で接続、弱信号やウォーム再起動では多数の拒否（実測 約35秒）または失敗。**完全に信頼できるのは純粋な WPA2-PSK のみ。**
2. **バンドステアリングが真の主犯**:メッシュがアソシエーション途中で弱いノードへ振り分け、拒否する（`status_code=16`;特徴 —— 拒否行の `bssid=` が接続したいノードと異なる、またはオールゼロ）。対策は **`roamoff=1`**（ファームのローミングエンジンを無効化）で、本ネットワークではこれだけで十分;**BSSID 固定は任意** —— 特定 AP を*保証*するだけで、代償は可搬性。

## 対処

**ルーター側** —— 2.4GHz の SSID を**純粋な WPA2-Personal** に（WPA3/混在をオフ）。**Wi-Fi 6 は有効のままで OK**（無害と実証済み —— 1 台の古い端末のためにネット全体を遅くしないこと）。

**クライアント側** —— 純粋 WPA2 + ファームのローミング無効化（`roamoff=1`）;BSSID 固定は任意:

```bash
# 1) WPA2-PSK を強制 + 起動時に無限リトライ（BSSID 固定なし —— roamoff がステアリングを処理）
sudo nmcli con modify "<YOUR_SSID_2G>" \
  802-11-wireless-security.key-mgmt wpa-psk \
  802-11-wireless-security.psk '<YOUR_WIFI_PASSWORD>' \
  802-11-wireless-security.pmf disable \
  connection.autoconnect yes connection.autoconnect-retries 0

# 2) ドライバ層:ファームのローミング無効化 —— これがバンドステアリングに勝つ本命
echo 'options brcmfmac roamoff=1' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo modprobe -r brcmfmac && sudo modprobe brcmfmac   # または再起動

# 3)（任意）強 AP を固定 —— roamoff だけで弱ノードに当たり続ける場合のみ
#    探す:sudo iw dev wlan0 scan | grep -iB5 'SSID: <YOUR_SSID_2G>' | grep -iE 'BSS|signal'
# sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid AA:BB:CC:DD:EE:FF
```

暗号化が本当に WPA2 のみ（SAE なし）かを確認:`sudo iw dev wlan0 scan | grep -iA22 'SSID: <YOUR_SSID_2G>' | grep Authentication` が `Authentication suites: PSK` であること —— `PSK SAE` では**ダメ**。

### ⚠️ 可搬性：固定 BSSID は AP のハードに紐づく

BSSID は**特定の AP 無線の MAC アドレス**です。固定はあくまで**メッシュのバンドステアリング専用**の回避策で、もろい：

- **ルーター/メッシュを変えると動かなくなる。** 新しい AP は別の MAC なので、固定した BSSID は存在せず、更新するまで Pi は決してアソシエートしない。
- 固定したノードが電源オフ/移設/故障すると、Pi は他ノードへ**ローミングしない**（BSSID 固定 + `roamoff=1` のため）。

ハードを変えたら、再スキャンして更新するだけ —— または固定解除：

```bash
# 新しい強ノードへ更新
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid <NEW_BSSID>
# または固定を完全解除（新ルーターが単一 AP・非メッシュならこちら）
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid ""
```

**常に固定が要るわけではありません。** バンドステアリングのあるメッシュ専用です。**単一 AP（非メッシュ）**なら、純粋 WPA2 + `roamoff=1` で大抵十分 —— BSSID 固定なし、可搬性も最良。最もきれいなのは **AP 側**:古い端末用にバンドステアリングを切った専用 SSID を用意すれば、クライアントは何も固定する必要がありません。

## さらに読む

- 📄 **[詳細トラブルシュート（日本語）](docs/troubleshooting.ja.md)** —— 対照実験、3 台比較、ファーム版数の分析、「効いた/効かなかった」、完全な出典リスト。
- 📄 他言語の詳細版：[English](docs/troubleshooting.en.md) · [繁體中文](docs/troubleshooting.zh-TW.md) · [한국어](docs/troubleshooting.ko.md)

## ⚠️ 苦労して得た警告

**このチップを高速な連続再接続や再起動ループでストレステストしないこと。** Zero 2 W では密な Wi-Fi 再アソシエーションが BCM43430 ファームを停止（halt）させ、**基板全体**を巻き込んでハングします（USB イーサも死に、電源を入れ直すしかない）。再接続の間隔が **10 秒以上なら安定**（実測 100/100）。このもろさは Zero 2 W の古いチップ固有 —— Pi 3B+/Pi 4（CYW43455 / BCM4345-6）は同じテストで 1000/1000 生き残ります。

## クレジット

実際の家庭内デプロイ(Xiaomi Wi-Fi 6 メッシュに 3 台の Raspberry Pi)の現場メモ。ホスト固有の秘密情報（Wi-Fi パスワード、SSID、BSSID、内部 IP）はすべてプレースホルダに置換済み。

## ライセンス

[MIT](LICENSE)
