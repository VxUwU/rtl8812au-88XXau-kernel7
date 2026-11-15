# RTL8812AU driver for Linux kernel 6.19 to 7.2

## What is this

This is the Realtek RTL8812AU Wi-Fi driver (the aircrack-ng `88XXau` one), fixed so it
actually builds and runs on recent Linux kernels. If you have an adapter like the
**Alfa AWUS036ACH** (USB id `0bda:8812`) and you are on a 2026-era kernel, the usual drivers
let you down in two ways:

* The driver that ships inside the kernel (`rtw88_8812au`) barely captures anything in
  monitor mode, and it crashes the moment you try packet injection with `aireplay-ng`.
* The popular aircrack-ng `88XXau` driver does injection fine, but it stopped compiling
  somewhere around kernel 6.15. On kernel 7 it fails with a pile of errors.

So this repo takes the aircrack-ng driver and adds the compatibility fixes it needs for the
newer kernels. Monitor mode and injection both work again. Nothing else about the driver
changes.

## Which kernels it works on

| Kernel | Compiles | Monitor + injection |
|--------|----------|---------------------|
| 6.19   | yes      | yes (where it started) |
| 7.1    | yes      | builds fine, not separately retested |
| 7.2    | yes      | yes, tested with `aireplay-ng --test` |

One source tree covers all of them. The kernel-specific fixes are wrapped in version checks,
so installing this will not break an older kernel you still boot.

## How to install

You need the kernel headers and the build tools first. On Debian or Kali:

```bash
sudo apt update
sudo apt install dkms build-essential linux-headers-$(uname -r)
```

Now get the driver onto your system. Clone the repo and copy it into the DKMS source folder:

```bash
git clone https://github.com/VxUwU/rtl8812au-88XXau-kernel7.git
cd rtl8812au-88XXau-kernel7
sudo cp -r . /usr/src/rtl8812au-88XXau-5.6.4.2-k7patch
```

Register and build it with DKMS. DKMS is what rebuilds the driver automatically every time
your kernel updates, so you only do this once:

```bash
sudo dkms add rtl8812au-88XXau/5.6.4.2-k7patch
sudo dkms build rtl8812au-88XXau/5.6.4.2-k7patch
sudo dkms install rtl8812au-88XXau/5.6.4.2-k7patch --force
```

Finally, tell the system to stop loading the weak built-in driver, and load this one:

```bash
echo -e 'blacklist rtw88_8812au\nblacklist rtw88_8812a' | sudo tee /etc/modprobe.d/blacklist-rtw88-8812au.conf
sudo modprobe -r rtw88_8812au
sudo modprobe 88XXau
```

If the adapter was already plugged in, unplug it and plug it back in so the right driver
takes over.

## How to check it worked

Replace `wlan0` with whatever your adapter is called (`ip link` will show you):

```bash
ethtool -i wlan0 | grep driver
```

You want to see `rtl88XXau`. Then put it in monitor mode and test injection:

```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 11
sudo aireplay-ng --test wlan0
```

If it says injection is working and lists nearby access points answering, you are done.

## Building without installing

If you just want to confirm it compiles against your running kernel without touching DKMS:

```bash
env -u KERNELRELEASE make -j$(nproc) KVER=$(uname -r) KSRC=/lib/modules/$(uname -r)/build
```

The `env -u KERNELRELEASE` part matters. Without it the driver's own Makefile goes down the
wrong path and the build fails. The DKMS config already handles this for you.

## What actually changed

Everything added on top of the original driver lives in the `kernel-patches` folder, as plain
diff files you can read. There are seven small fixes in total.

Three of them were needed from kernel 6.16 onward:

1. The build system stopped honoring the old `EXTRA_CFLAGS` setting, so the driver lost all
   its include paths and defines. Switched to `ccflags-y`.
2. The kernel renamed two timer functions. Added a small compatibility shim.
3. Some Wi-Fi control callbacks gained an extra `radio_idx` argument.

Four more were needed for kernel 7:

4. Turned off a PPPoE bridging feature that no longer compiles and that a monitor or
   injection setup never uses anyway.
5. The kernel removed `strncpy`. Added a drop-in replacement.
6. A batch of Wi-Fi callbacks changed one of their arguments from `net_device` to
   `wireless_dev`. Updated the nine affected ones plus two related notifications.
7. One more callback (`remain_on_channel`) gained an extra argument in kernel 7.2.

If you would rather apply these to a fresh copy of the upstream driver yourself:

```bash
git clone https://github.com/aircrack-ng/rtl8812au
cd rtl8812au
sed -i '/BUILD_EXCLUSIVE/d' dkms.conf
patch -p1 < ../kernel-patches/rtl8812au-88XXau-kernel6.19.patch
patch -p1 < ../kernel-patches/rtl8812au-88XXau-kernel7.patch
```

## If a future kernel breaks it again

Build it against the new kernel, read whatever errors come up, and add the same kind of
version check around each fix. The older kernels keep working because every change is guarded
by the kernel version it applies to. This is roughly how fixes 4 through 7 were made.

## Credits and license

The driver itself comes from Realtek and the
[aircrack-ng rtl8812au project](https://github.com/aircrack-ng/rtl8812au). All the real work
is theirs. This repo only adds the kernel compatibility fixes. It stays under the same GPLv2
license, in the `LICENSE` file. The original project README is kept here as
`README.upstream.md`.
