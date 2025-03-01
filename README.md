# What

This repo currently contains a small tool for flipping an M.2 XMM7360 modem,
like the Fibocom L850-GL, from PCIe into USB mode.
This allows you to talk to it with `mbimcli`, `ModemManager` etc.

# Usage

This script needs [the acpi_call kernel module](https://github.com/mkottman/acpi_call).
You can build and install it like this:

```
git clone https://github.com/mkottman/acpi_call.git
cd acpi_call
make && sudo make install
```

Then, you can simply run this script:

```
sudo ./xmm2usb
```

After 5-10 seconds, your new USB device should appear.

# MBIM Switch

My modem came up initially as a 3x ACM + 3x NCM USB device.
To switch to MBIM mode I ran:

```
sudo screen /dev/ttyACM0
```

and issued

```
AT+GTUSBMODE?
AT+GTUSBMODE=7
AT+CFUN=15
```

Take note of the USB mode reported after the first command in case you want to put it back later...!

(There is an old L8 family AT command doc floating around saying that mode 2 is MBIM. It does not apply to this modem.)

# How does it work?

According to the modem docs, the chipset tries talking via PCIe first, then goes to USB if it doesn't succeed.
Luckily for us, we have two moving parts here we can use:
we can talk to the upstream PCI Express port to disable the link,
and then we can use ACPI to hit the modem's reset line so that it starts looking anew.
That's all this script does.

The modem seems to happily stay in USB mode across a suspend/resume, but the PCIe link needs to be disabled on each resume.
It'd be cute to have a PCI stub driver to do this, I suppose.

# "FCC Lock"

My modem, in my Thinkpad T490, initially seems to be stuck in flight mode (`AT+CFUN?` returns `+CFUN: 4,0` and can't be changed).

This turns out to be a mechanism the authors call "FCC Lock".
This prevents the radoi from being enabled until it is unlocked;
this is the purpose of the `ModemAuthenticator.exe` which comes with the driver in Windows.
This has the effect of tying the modem to particular machines:
the unlock key is stored in the system's SMBIOS.

This looks very much like it has been done for regulatory purposes:
these days you have to test the software, radio, and antennas together in the final product to meet regulatory requirements.
You need a lot of equipment, time, and people who know what they are doing, so this is expensive.
Also, porting a PCI driver for this thing would suck, because it's actually quite complicated to talk to.
Thus Lenovo only tested the Windows configuration and implemented this locking mechanism to maintain compliance.

**Regulation is a good thing here - this is why we so rarely have problems with interference,
and tries to make sure we don't get cooked by dodgy gear.**
Please consider this carefully before you decide to unlock your modem.

I carefully reverse engineered the unlock challenge/response sequence before realising that you can permanently bypass it with just a couple of AT commands.

**_This should not brick your device, but I make no guarantees. Here be dragons!_**

You can unlock the modem by issuing

```
at@nvm:fix_cat_fcclock.fcclock_mode=0
```

(The default mode, at least on my modem, is 2.)

This will work until the modem is next power cycled or reset. To make the change permanent, issue:

```
at@store_nvm(fix_cat_fcclock)
```

The modem will then power up with radio enabled in future.

# PCI

I did start doing some reverse engineering and writing a PCI driver, just for fun.
So far you can send AT commands to a couple of ports.
I haven't figured out where to stick an MBIM command to get a response though.
