# Raspberry Pi Zero 2 W — Wi-Fi `ASSOC-REJECT status_code=16` on a mesh network: root cause & fix

**Languages:** **English** · [繁體中文](README.zh-TW.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

A field write-up of why a **Raspberry Pi Zero 2 W** (Broadcom **BCM43430** / `brcmfmac` driver) repeatedly fails to associate with a **Wi-Fi mesh** router — `CTRL-EVENT-ASSOC-REJECT ... status_code=16` — and what actually fixed it. Includes a controlled experiment isolating each variable, a 3-device comparison, and a clear list of which conclusions are backed by upstream issues vs. measured first-hand.

> **TL;DR** — On an old BCM43430 chip:
> 1. **Use pure WPA2-PSK.** Pure WPA3/SAE *always* fails on this chip; WPA2/WPA3 *mixed mode* will connect but is flaky (fine on a strong-signal cold boot, unreliable on weak signal / warm reboot). **⚠️ Since `wpasupplicant 2:2.10-24+rpt3` (2026-11), mixed mode doesn't connect at all — see the [2026-11 update](#2026-11-update) below.**
> 2. **`options brcmfmac roamoff=1`** in `/etc/modprobe.d/brcmfmac.conf` — disables the firmware roaming engine, which is what **mesh band steering** trips over. On our network this alone reliably connects and self-selects the strong AP on ~88% of cold boots — **no BSSID pin needed**, and it stays portable.
> 3. **(Optional) pin one strong AP's BSSID** only if you must *guarantee* the strong AP every boot — it works but is brittle (breaks when you change routers).
>
> **The Wi-Fi generation (6/7) is NOT the culprit** (a common misdiagnosis — proven below). **Do not downgrade firmware blindly** — the popular "downgrade to 9.88.4.65" advice targets a *different chip revision* and does not apply to BCM43430/1.

<a id="2026-11-update"></a>
## ⚠️ 2026-11 update: `wpasupplicant 2:2.10-24+rpt3` breaks WPA2/WPA3 mixed mode

After a routine `apt full-upgrade` on Raspberry Pi OS (trixie), `wpasupplicant` went from `2:2.10-24` to **`2:2.10-24+rpt3`** — and the Zero 2 W **stopped connecting to the mixed-mode SSID entirely** (same network it had been connecting to for months):

```
wpa_supplicant: wlan0: Authentication with <BSSID> timed out.
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... reason=CONN_FAILED
NetworkManager: device (wlan0): Activation: (wifi) association took too long
NetworkManager: device (wlan0): state change: config -> failed (reason 'no-secrets')
```

After that, NetworkManager **stops auto-retrying** (even with `autoconnect-retries 0`), so Wi-Fi stays down until you bring it up by hand.

- **Likely cause:** the `+rpt3` changelog says *"Add upstream patch to fix reporting of SAE support over D-Bus"*. NetworkManager now learns the supplicant can do SAE and uses SAE (WPA3) on a mixed-mode AP — which this chip can't do. Before, it fell back to PSK. (Inferred from the changelog; what we verified is that downgrading alone fixed it instantly.)
- **Fix A (recommended): set the router to pure WPA2-PSK.** Same as the TL;DR. Nothing to hold, future upgrades stay safe.
- **Fix B: downgrade and hold** (if you must keep mixed mode):
  ```bash
  sudo apt-get install --allow-downgrades wpasupplicant=2:2.10-24
  sudo apt-mark hold wpasupplicant
  sudo nmcli con up "<YOUR_SSID_2G>"   # NM won't retry by itself after 'no-secrets'
  ```
  Result: connected on the first try with 0 rejects, and still auto-connects after rebooting into the new kernel. Trade-off: while held, `wpasupplicant` gets no security updates — run `sudo apt-mark unhold wpasupplicant` once the router is on pure WPA2 (or a fixed package ships).
- **Not affected:** the same upgrade bumped `firmware-brcm80211` (20260519), but the file this board actually loads (`brcmfmac43436s-sdio.bin`) is still `7.45.96.s1` — no Wi-Fi firmware change. Check yours with `journalctl -k -b | grep 'Firmware: BCM'`.

---

## Hardware note (does this apply to a stock Zero 2 W?)

**Yes — and it matters *more* if your board is stock.** A stock Raspberry Pi Zero 2 W is **Wi-Fi only**: it has no Ethernet, so Wi-Fi is your *only* link and the fix below is essential.

This write-up comes from a board fitted with a **USB Ethernet HAT** (Waveshare ETH/USB HUB, Realtek **RTL8152** 10/100), so it also has a wired `eth0`. That lets the deployment run **wired-primary, Wi-Fi-as-backup** — the most robust arrangement, but **one most readers won't have**. If you do add such a HAT, note the wedge caveat in the warning below: the hub draws 5 V from the Pi, and a brownout can drop the whole USB bus (Ethernet included).

## The symptom

```
wpa_supplicant: wlan0: CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16
wpa_supplicant: wlan0: CTRL-EVENT-SSID-TEMP-DISABLED ... auth_failures=N reason=CONN_FAILED
```

- `nmcli con up <SSID>` fails repeatedly and eventually reports
  `Error: ... Secrets were required, but not provided` — this is NetworkManager **mis-reporting** a password problem after many failed associations. **The password is not wrong.**
- `bssid=00:00:00:00:00:00` (all zeros) means the **firmware rejected the association before the 4-way handshake** → it's an AP encryption/mode incompatibility, not a credential issue.

## Root cause

The Zero 2 W's Wi-Fi chip is an old **BCM43430** (firmware `7.45.96.s1`, 2023). Two things matter on a `*_2.4G` SSID on a consumer mesh:

1. **Encryption.** Pure WPA3/SAE *always* makes the old firmware reject the association. WPA2/WPA3 *mixed mode* **can** associate (the client negotiates PSK over SAE) but is **unreliable** — on a strong signal it connects in ~1–2 s, on a weak signal or a warm reboot it takes many rejects (≈35 s seen) or fails. **Pure WPA2-PSK is the only fully reliable option.** (Mixed mode working at all depends on `wpasupplicant` ≤ `2:2.10-24` — see the [2026-11 update](#2026-11-update).)
2. **Band steering** is the real culprit: the mesh hands the client off to a weaker node mid-association and rejects it (`status_code=16`; tell-tale sign — the `bssid=` in the reject line differs from the node you intended, or is all-zeros). The fix is **`roamoff=1`** (disable the firmware roaming engine), which on our network is enough on its own; **pinning a BSSID is optional** — it only *guarantees* a specific AP, at the cost of portability.

## The fix

**Router side** — set the 2.4 GHz SSID to **pure WPA2-Personal** (turn off WPA3/mixed). **Wi-Fi 6/7 can stay on** (verified harmless — don't slow your whole network down for one old client). (Our test network is actually Wi-Fi 7; an 802.11n client ignores the AP's HE/EHT either way.)

**Client side** — pure WPA2 + disable firmware roaming (`roamoff=1`); a BSSID pin is optional:

```bash
# 1) Force WPA2-PSK + autoconnect forever (no BSSID pin — roamoff handles steering)
sudo nmcli con modify "<YOUR_SSID_2G>" \
  802-11-wireless-security.key-mgmt wpa-psk \
  802-11-wireless-security.psk '<YOUR_WIFI_PASSWORD>' \
  802-11-wireless-security.pmf disable \
  connection.autoconnect yes connection.autoconnect-retries 0

# 2) Driver-level: disable firmware roaming — THIS is what beats band steering
echo 'options brcmfmac roamoff=1' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo modprobe -r brcmfmac && sudo modprobe brcmfmac   # or reboot

# 3) (Optional) pin the strong AP — only if roamoff alone keeps landing on a weak node
#    Find it: sudo iw dev wlan0 scan | grep -iB5 'SSID: <YOUR_SSID_2G>' | grep -iE 'BSS|signal'
# sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid AA:BB:CC:DD:EE:FF
```

Verify the encryption is truly WPA2-only (no SAE): `sudo iw dev wlan0 scan | grep -iA22 'SSID: <YOUR_SSID_2G>' | grep Authentication` must read `Authentication suites: PSK` — **not** `PSK SAE`.

### ⚠️ Portability: a pinned BSSID is tied to your AP's hardware

A BSSID **is the MAC address of one specific AP radio**. Pinning it is a workaround for **mesh band steering only** — and it is brittle:

- **Change your router / mesh and it stops working.** The new AP has a different MAC, so the pinned BSSID no longer exists and the Pi will never associate until you update it.
- If the pinned node is powered off / moved / fails, the Pi **won't roam** to another node (you pinned it *and* set `roamoff=1`).

When you change hardware, just re-scan and update — or unpin:

```bash
# update to the new strong node
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid <NEW_BSSID>
# or remove the pin entirely (do this if the new router is a single AP, not a mesh)
sudo nmcli con modify "<YOUR_SSID_2G>" 802-11-wireless.bssid ""
```

**You don't always need the pin.** It's only for band-steering mesh networks. With a **single AP (non-mesh)**, pure WPA2 + `roamoff=1` is usually enough — no BSSID pin, fully portable. The cleanest fix of all is **AP-side**: give the old client a dedicated SSID with band steering off, so the client never needs to pin anything.

## Read more

- 📄 **[Full troubleshooting write-up (English)](docs/troubleshooting.en.md)** — the controlled experiment, the 3-device comparison, the firmware-version analysis, the "what worked / what didn't", and the full source list.
- 📄 Detailed write-up in [繁體中文](docs/troubleshooting.zh-TW.md) · [日本語](docs/troubleshooting.ja.md) · [한국어](docs/troubleshooting.ko.md)

## ⚠️ One hard-won warning

**Do not stress-test this chip with rapid back-to-back reconnects or reboot loops.** On the Zero 2 W, dense Wi-Fi re-association can wedge the BCM43430 firmware and hang the *entire board* (even USB Ethernet dies; only a power-cycle recovers it). Reconnects spaced **≥10 s apart are stable** (measured 100/100). This fragility is specific to the Zero 2 W's old chip — a Pi 3B+/Pi 4 (CYW43455 / BCM4345-6) survives the same test 1000/1000.

## Credits

Field notes from a real home deployment (three Raspberry Pi boards on a Xiaomi Wi-Fi 7 mesh). All host-specific secrets (Wi-Fi password, SSID, BSSID, internal IPs) have been replaced with placeholders.

## License

[MIT](LICENSE)
