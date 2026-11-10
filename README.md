# Linux Wi-Fi Stack Lab

A hands-on exploration of the Linux wireless stack using `mac80211_hwsim` on Xubuntu.

## Objective

Simulate Wi-Fi hardware and create a virtual Access Point (AP) with a connected Wi-Fi client, without requiring a second physical Wi-Fi adapter.

## Environment

- OS: Xubuntu
- Kernel: Linux 6.17
- Architecture: x86_64
- Wireless simulator: `mac80211_hwsim`

## Tools

- `iw` – inspect and configure wireless interfaces
- `hostapd` – configure the Access Point
- `wpa_supplicant` – connect the Wi-Fi client
- `nl80211` – userspace interface to the Linux wireless subsystem

## 1. Install dependencies

```bash
sudo apt update
sudo apt install iw hostapd wpasupplicant
```

Check that the simulator is available:

```bash
modinfo mac80211_hwsim
```

## 2. Create virtual Wi-Fi radios

Load the simulator with two virtual radios:

```bash
sudo modprobe mac80211_hwsim radios=2
```

Inspect the wireless devices:

```bash
iw dev
iw phy
```

Identify the simulated radios:

```bash
for p in /sys/class/ieee80211/phy*; do
    echo "===== $p ====="
    readlink -f "$p/device/driver"
done
```

## 3. Create the AP and client interfaces

Replace `phy1` and `phy2` if your PHY numbers are different.

```bash
sudo iw phy phy1 interface add hwsim-ap type __ap
sudo iw phy phy2 interface add hwsim-sta type managed

sudo ip link set hwsim-ap up
sudo ip link set hwsim-sta up
```

## 4. Configure the Access Point

The AP uses WPA2-Personal (WPA2-PSK).

Example `hostapd.conf`:

```ini
interface=hwsim-ap
driver=nl80211
ssid=ENSI-WIFI-LAB
hw_mode=g
channel=6
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=CHANGE_ME_12345
```

Start the AP:

```bash
sudo hostapd hostapd.conf
```

## 5. Configure the client

Example `wpa_supplicant.conf`:

```ini
ctrl_interface=DIR=/run/wpa_supplicant-hwsim
update_config=0
p2p_disabled=1
ap_scan=1

network={
    ssid="ENSI-WIFI-LAB"
    psk="CHANGE_ME_12345"
    key_mgmt=WPA-PSK
}
```

Use the same password as the AP.

Start the client:

```bash
sudo mkdir -p /run/wpa_supplicant-hwsim

sudo wpa_supplicant \
    -D nl80211 \
    -i hwsim-sta \
    -c wpa_supplicant.conf
```

## 6. Verify the connection

Check the client connection:

```bash
iw dev hwsim-sta link
```

Inspect the connected stations on the AP:

```bash
sudo iw dev hwsim-ap station dump
```

### Observed results

The experiment successfully demonstrated:

- AP initialization (`AP-ENABLED`)
- Client authentication and association
- WPA2 pairwise key handshake completion
- Station authorization

The client connected to the virtual AP on channel 6 (2437 MHz).

## 7. Explore the kernel wireless stack

Inspect loaded modules:

```bash
lsmod | grep -E 'mac80211|cfg80211|hwsim'
```

Inspect kernel messages:

```bash
sudo dmesg | grep -iE 'hwsim|cfg80211|mac80211|wlan'
```

Inspect kernel configuration:

```bash
grep -E 'CONFIG_(CFG80211|MAC80211|MAC80211_HWSIM)' \
    /boot/config-$(uname -r)
```

The three main components were configured as loadable kernel modules (`=m`).

Observe wireless events:

```bash
sudo iw event -t
```

## Linux wireless architecture

```text
Userspace
  |
  +-- iw
  +-- hostapd
  +-- wpa_supplicant
  |
nl80211
  |
cfg80211
  |
mac80211
  |
mac80211_hwsim
  |
Virtual Wi-Fi radios
```

| Component | Responsibility |
|---|---|
| `nl80211` | Netlink interface for wireless configuration |
| `cfg80211` | Kernel wireless configuration layer |
| `mac80211` | SoftMAC wireless framework |
| `mac80211_hwsim` | Simulated wireless hardware |

## Notes

A kernel warning was observed during scanning in `ieee80211_is_radio_idx_in_scan_req()` (`net/mac80211/util.c`). It has not yet been investigated.

This lab focuses on wireless configuration, simulation, and Linux wireless architecture. It does not implement a Wi-Fi driver.