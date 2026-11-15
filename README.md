# rtl8812au-88XXau — patched for Linux kernel 6.19 → 7.2

A fork of the aircrack-ng [`rtl8812au`](https://github.com/aircrack-ng/rtl8812au) (`88XXau`)
driver for **RTL8812AU** USB adapters (e.g. **Alfa AWUS036ACH**, USB ID `0bda:8812`),
patched so it **builds and injects on modern kernels** — verified on `6.19`, `7.1` and `7.2`.

The in-kernel `rtw88_8812au` driver delivers almost no frames in monitor mode and crashes
under `aireplay-ng`, and the stock aircrack `88XXau` no longer compiles past kernel ~6.15.
This fork fixes the compile breaks up to kernel 7.2 while keeping monitor mode + frame
injection working.

> **License:** GPLv2 (unchanged from upstream — see [`LICENSE`](LICENSE)). All credit for the
> driver itself goes to Realtek and the aircrack-ng maintainers; this fork only adds kernel
> compatibility patches. Upstream's own README is kept as
> [`README.upstream.md`](README.upstream.md).

## Kernel compatibility

| Kernel | Builds | Monitor + injection |
|---|---|---|
| `6.19.x` | ✅ | ✅ (originally verified) |
| `7.1.x`  | ✅ | — (builds; not separately injection-tested) |
| `7.2.x`  | ✅ | ✅ **verified** (`aireplay-ng --test` → APs answering 60–100%) |

Everything is `KERNEL_VERSION`-guarded, so the **same source compiles across all three** —
older kernels keep working.

## Install (DKMS — recommended)

```bash
sudo apt install -y dkms build-essential "linux-headers-$(uname -r)"

# drop the source into /usr/src and register with DKMS
sudo cp -r . /usr/src/rtl8812au-88XXau-5.6.4.2-k7patch
sudo dkms add     -m rtl8812au-88XXau -v 5.6.4.2-k7patch
sudo dkms build   -m rtl8812au-88XXau -v 5.6.4.2-k7patch
sudo dkms install -m rtl8812au-88XXau -v 5.6.4.2-k7patch --force

# make sure the in-kernel driver never wins
echo -e 'blacklist rtw88_8812au\nblacklist rtw88_8812a' \
  | sudo tee /etc/modprobe.d/blacklist-rtw88-8812au.conf
sudo modprobe -r rtw88_8812au 2>/dev/null; sudo modprobe 88XXau
```

DKMS auto-rebuilds the module on every kernel update.

### Verify

```bash
ethtool -i wlan0 | grep driver        # -> rtl88XXau   (adjust iface name)
sudo ip link set wlan0 down && sudo iw dev wlan0 set type monitor && sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 11
sudo tcpdump -i wlan0 -nn -c 20        # frames flow
sudo aireplay-ng --test wlan0          # "Injection is working!"
```

### Quick scratch build (no install)

```bash
env -u KERNELRELEASE make -j"$(nproc)" KVER="$(uname -r)" KSRC="/lib/modules/$(uname -r)/build"
```

> **Note:** the DKMS `MAKE` line uses `env -u KERNELRELEASE …` on purpose — without it, the
> driver's dual-purpose Makefile takes the wrong branch and fails on `phydm.mk`.

## What was patched

On top of the upstream driver, **7 patches** (all in [`kernel-patches/`](kernel-patches/)):

**Kernel 6.16+ (3):**
1. `Makefile`: `EXTRA_CFLAGS` → `ccflags-y` (kbuild dropped `EXTRA_CFLAGS`).
2. Timer API rename compat: `from_timer`→`timer_container_of`, `del_timer_sync`→`timer_delete_sync`.
3. cfg80211 ops gained an `int radio_idx` param (`set_wiphy_params`, `set/get_tx_power`).

**Kernel 7.x (4):**
4. Disable `CONFIG_BR_EXT` — kernel 7 removed `pppoe_hdr.tag` / `pppoe_tag.tag_data`; the
   PPPoE bridge helper is irrelevant to a monitor/injection radio.
5. `strncpy()` shim — `strncpy` was removed from the kernel; added a padding-compatible
   fallback (guarded `≥ 7.0`).
6. `struct cfg80211_ops`: 2nd arg `net_device *` → `wireless_dev *` on 9 key/station
   callbacks + the `cfg80211_new_sta`/`del_sta` notify calls (kernel 7 per-link/MLO rework).
   `set_default_key` is **not** changed (still `net_device *`).
7. `remain_on_channel` gained a trailing `const u8 *rx_addr` (guarded `≥ 7.2`).

To apply the diffs onto a fresh upstream clone instead:

```bash
git clone https://github.com/aircrack-ng/rtl8812au && cd rtl8812au
sed -i '/BUILD_EXCLUSIVE/d' dkms.conf
patch -p1 < /path/to/kernel-patches/rtl8812au-88XXau-kernel6.19.patch
patch -p1 < /path/to/kernel-patches/rtl8812au-88XXau-kernel7.patch
```

## When a future kernel breaks the build

Rebuild against it (`env -u KERNELRELEASE make KVER=<ver> KSRC=/lib/modules/<ver>/build`),
read the new errors, and add `#if LINUX_VERSION_CODE >= KERNEL_VERSION(x,y,0)` guards the
same way — older kernels keep compiling.

## Credits

- [aircrack-ng/rtl8812au](https://github.com/aircrack-ng/rtl8812au) — the upstream driver.
- Realtek — the original source.
- Kernel 6.19 → 7.2 compatibility patches in this fork.
