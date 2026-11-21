# X96 Max Plus Ultra — Working Infrared (IR) Receiver/Transmitter

Verified on hardware (X96 Max Plus Ultra, Amlogic S905X4 / SC2, kernel
`6.18.59-meson64`): the bundled IR remote decodes NEC scancodes on
`/dev/input/event0` via `rc0`, and `meson-ir-tx` provides IR transmit.

## How it works

| Piece | Source |
|---|---|
| `meson-ir.ko` | mainline `drivers/media/rc/meson-ir.c`, patched to write the vendor IRQ control value `0xCFA10BB8` so the SC2 IR decoder asserts its interrupt |
| `meson-ir-tx.ko` | mainline IR transmitter |
| `rc-core`, decoders, `rc-x96max` | mainline RC subsystem + X96 keymap |
| Device tree | `ir@84040` node, `compatible = "amlogic,meson-s4-ir"`, `linux,rc-map-name = "rc-x96max"` |
| Autoload | `/etc/modules-load.d/infrared.conf` → `meson-ir`, `meson-ir-tx` |

Without the patched `meson-ir.ko`, `rc0` exists but stays silent — the
SC2 block needs the vendor IRQ register write. The DTB in the release
contains the working `ir@84040` node.

### Hardware decoder must be armed

`meson-s4-ir` uses the SC2 hardware decoder, which stays disabled until
userspace selects a protocol. Without it `rc0` registers but the IRQ
never fires and `ir-keytable -t` shows nothing. Enable NEC once per boot:

```sh
echo nec > /sys/class/rc/rc0/protocols
# or: ir-keytable -s rc0 -p nec
```

Armbian images built from `AmakerGame/amlogic-s9xxx-armbian` ship an
enabled `ir-nec.service` that does this automatically after `rc0`
appears. Verified live: NEC scancodes (e.g. `0x116`) decode on
`/dev/input/event0`.

## Releases

See [Releases](../../releases) for `x96-ir-6.18.59-meson64.tar.gz`,
the matching `meson-sc2-x96-max-plus-ultra.dtb`, `infrared.conf` and
`sha256sums.txt`.

## Install (Armbian / amlogic-s9xxx-armbian)

The `kernel_s905x4` kernel packages in
[AmakerGame/amlogic-s9xxx-armbian](https://github.com/AmakerGame/amlogic-s9xxx-armbian)
already contain all IR modules and the IR-enabled DTB — nothing extra is
required for images built from that channel.

For a manual install:

```sh
sudo tar -xzf x96-ir-6.18.59-meson64.tar.gz -C /
sudo cp meson-sc2-x96-max-plus-ultra.dtb /boot/dtb/amlogic/
sudo cp infrared.conf /etc/modules-load.d/
sudo depmod -a && sudo reboot
```

## Verify

```sh
ls /sys/class/rc/                 # rc0
cat /sys/class/rc/rc0/protocols   # ... nec ...
ir-keytable -t                    # press remote buttons -> scancodes
```
