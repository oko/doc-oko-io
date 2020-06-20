# OVMF

## Building OVMF

```
$ sudo yum groupinstall 'Development tools'
$ sudo yum install libuuid-devel iasl nasm
$ git clone https://github.com/tianocore/edk2.git
$ cd edk2
$ git submodule init && git submodule update
$ make -C BaseTools
$ . edksetup.sh
$ cat <<-EOF > Conf/target.txt
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = RELEASE
TARGET_ARCH           = X64
TOOL_CHAIN_CONF       = Conf/tools_def.txt
TOOL_CHAIN_TAG        = GCC48
BUILD_RULE_CONF = Conf/build_rule.txt
EOF
$ build -D NETWORK_IP6_ENABLE=TRUE -D HTTP_BOOT_ENABLE=TRUE -D TLS_ENABLE=TRUE
```

If you're on Ubuntu, you'll need to do:

```
sudo apt install uuid-dev iasl nasm gcc5
cat <<-EOF > Conf/target.txt
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = RELEASE
TARGET_ARCH           = X64
TOOL_CHAIN_CONF       = Conf/tools_def.txt
TOOL_CHAIN_TAG        = GCC5
BUILD_RULE_CONF = Conf/build_rule.txt
EOF
```

This sets things up for a GCC5 build instead.

## Running with OVMF

Script to run OVMF on QEMU:

```
#!/bin/bash
set -eux
eficode="$1"
efivars="$2"

wd="$(mktemp -d)"
vars="$wd/$(basename "$efivars")"

cp "$efivars" "$vars"

qemu-system-x86_64 \
    -name guest=ovmftest \
    -enable-kvm -cpu host \
    -nographic \
    -drive if=pflash,format=raw,readonly,file="$eficode" \
    -drive if=pflash,format=raw,file="$vars" \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-pci,romfile=,netdev=net0 -net none
```

Use `screen /dev/pts/XX` based on the output of the QEMU script at startup to connect to serial console.

## Comparing VARS.fd files

1. Start up a VM with a fresh `OVMF_VARS.fd` using the above script
2. Shut down the VM
3. Make a copy of the `OVMF_VARS.fd` file from the temporary folder used by the script
4. Start up another VM with the copied `OVMF_VARS.fd`
5. Wait for setup to launch, tweak things, and shut down the VM
6. Repeat (3)
7. Dump both the (3) and (6) files to hex files using `xxd FILE.fd > FILE.hex`
8. `diff` the hex files

	```
	977c977
	< 0003d00: 0000 7f57 d35b f9ec 0200 2400 afaf 0400  ...W.[....$.....
	---
	> 0003d00: 0000 7e56 e533 e915 0200 2400 afaf 0400  ..~V.3....$.....
	985c985
	< 0003d80: 0000 7f57 d35b 7080 0300 3400 afaf 0800  ...W.[p...4.....
	---
	> 0003d80: 0000 7e56 e533 5fa9 0300 3400 afaf 0800  ..~V.3_...4.....
	994c994
	< 0003e10: 0001 0001 2496 90cc 5254 0012 3456 ffff  ....$...RT..4V..
	---
	> 0003e10: 0001 0001 2496 8c23 5254 0012 3456 ffff  ....$..#RT..4V..
	1055c1055
	< 00041e0: 9f59 4d85 0ee2 1a52 2c59 b2ff aa55 3f00  .YM....R,Y...U?.
	---
	> 00041e0: 9f59 4d85 0ee2 1a52 2c59 b2ff aa55 3c00  .YM....R,Y...U<.
	1087,1088c1087,1088
	< 00043e0: 3100 3200 3300 3400 3500 3600 0000 7f57  1.2.3.4.5.6....W
	< 00043f0: d35b 6f80 0300 3400 afaf 0800 0000 0100  .[o...4.........
	---
	> 00043e0: 3100 3200 3300 3400 3500 3600 0000 7e56  1.2.3.4.5.6...~V
	> 00043f0: e533 5ea9 0300 3400 afaf 0800 0000 0100  .3^...4.........
	1091,1097c1091,1097
	< 0004420: 0000 5054 00ff fe12 3456 ffff ffff ffff  ..PT....4V......
	< 0004430: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	< 0004440: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	< 0004450: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	< 0004460: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	< 0004470: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	< 0004480: ffff ffff ffff ffff ffff ffff ffff ffff  ................
	---
	> 0004420: 0000 5054 00ff fe12 3456 ffff aa55 3f00  ..PT....4V...U?.
	> 0004430: 0700 0000 0000 0000 0000 0000 0000 0000  ................
	> 0004440: 0000 0000 0000 0000 0000 0000 0000 0000  ................
	> 0004450: 1400 0000 0a00 0000 61df e48b ca93 d211  ........a.......
	> 0004460: aa0d 00e0 9803 2b8c 4200 6f00 6f00 7400  ......+.B.o.o.t.
	> 0004470: 4f00 7200 6400 6500 7200 0000 0300 0200  O.r.d.e.r.......
	> 0004480: 0100 0400 0000 ffff ffff ffff ffff ffff  ................
	```

	`$ diff <(xxd ~/OVMF_VARS_v6.fd) <(xxd ~/OVMF_VARS_v6_test.fd)`

## Intel E1000 Drivers

Download link: https://downloadmirror.intel.com/19186/eng/PREBOOT.EXE

From EDK2 docs for ease of recall:

```
* Also independently of the iPXE NIC drivers, Intel's proprietary E1000 NIC
  driver (from the BootUtil distribution) can be embedded in the OVMF image at
  build time:

  - Download BootUtil:
    - Navigate to
      https://downloadcenter.intel.com/download/19186/Ethernet-Intel-Ethernet-Connections-Boot-Utility-Preboot-Images-and-EFI-Drivers
    - Click the download link for "PREBOOT.EXE".
    - Accept the Intel Software License Agreement that appears.
    - Unzip "PREBOOT.EXE" into a separate directory (this works with the
      "unzip" utility on platforms different from Windows as well).
    - Copy the "APPS/EFI/EFIx64/E3522X2.EFI" driver binary to
      "Intel3.5/EFIX64/E3522X2.EFI" in your WORKSPACE.
    - Intel have stopped distributing an IA32 driver binary (which used to
      match the filename pattern "E35??E2.EFI"), thus this method will only
      work for the IA32X64 and X64 builds of OVMF.

  - Include the driver in OVMF during the build:
    - Add "-D E1000_ENABLE" to your build command (only when building
      "OvmfPkg/OvmfPkgIa32X64.dsc" or "OvmfPkg/OvmfPkgX64.dsc").
    - For example: "build -D E1000_ENABLE".
```

