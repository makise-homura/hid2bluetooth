# HID to Bluetooth bridge

Lets you use your computer as Bluetooth converter for your keyboard and mouse.
Might be an useful thing to control your tablet or laptop from another computer.

Based on code from [keyboard_mouse_emulate_on_raspberry](https://github.com/quangthanh010290/keyboard_mouse_emulate_on_raspberry),
but completely revamped to be a standalone Python script rather than complex system
relying on D-Bus, tmux and other stuff with a massive bunch of requirements.

## Use cases

There are several devices like [these ones](https://aliexpress.ru/wholesale?catId=202004372&SearchText=bluetooth keyboard mouse converter)
on AliExpress, but they cost $20 to $30. In the other case, you may have an SBC like
Raspberry Pi or Orange Pi that you use as headless home server. So why not implement
such a functionality for free using any cheap keyboard and mouse and already existing
SBC--for free?

For example, I have a tablet with broken touch screen panel, and I'd like to use it
as a stationary video/music player with keyboard and mouse are connected to it. But
it can't utilize simultaneous USB OTG and charging, so reconnecting cables may be
a burden. But what if keyboard and mouse are connected through bluetooth rather
than USB? System looks enough perfect in this case: you have USB charging, audio
output connected to amplifier with speakers, and input devices are used through
bluetooth to select video or music to play. And everything works quite well with
the SBC that is a home server at the same time.

# Installation

Just copy `hid2bluetooth` to `/usr/sbin` or wherever convenient. Set executable permissions if needed.

# Usage

```
hid2bluetooth [-h] [-b DEV] [-k DEV] [-m DEV] [-v] [-V]
```

Usually you would specify only `-b`, `-k`, and `-d` arguments with corresponding values.

## Arguments

* `-h`, `--help`: show help message and exit
* `-V`, `--version`: show program's version number and exit
* `-b DEV`, `--bluetooth DEV`: bluetooth device to use (example and default: hci0)
* `-k DEV`, `--keyboard DEV`: keyboard event device to use (example and default: event0)
* `-m DEV`, `--mouse DEV`: mouse event device to use (example and default: event1)
* `-g`, `--dontgrab`: don't grab devices on start, allowing them to source inputs for other clients
* `-v`, `--verbose`: log events: specify once for output, twice for input, thrice for both

## Determining device names

### Input (event) devices

You may list input event devices by the following command:

```
ls /dev/input/by-id/ -la
```

It will result with something like this:

```
total 0
drwxr-xr-x 2 root root 220 Dec  8 21:26 .
drwxr-xr-x 4 root root 300 Dec  8 21:26 ..
lrwxrwxrwx 1 root root   9 Oct  4 12:23 usb-0c45_USB_Keyboard-event-if01 -> ../event3
lrwxrwxrwx 1 root root   9 Oct  4 12:23 usb-0c45_USB_Keyboard-event-kbd -> ../event1
lrwxrwxrwx 1 root root   9 Oct  4 12:23 usb-0c45_USB_Keyboard-if01-event-kbd -> ../event4
lrwxrwxrwx 1 root root   9 Dec  8 21:26 usb-A4TECH_USB_Device-event-if00 -> ../event6
lrwxrwxrwx 1 root root   9 Dec  8 21:26 usb-A4TECH_USB_Device-event-kbd -> ../event5
lrwxrwxrwx 1 root root   9 Dec  8 21:26 usb-A4TECH_USB_Device-if01-event-mouse -> ../event7
lrwxrwxrwx 1 root root   9 Dec  8 21:26 usb-A4TECH_USB_Device-if01-mouse -> ../mouse1
lrwxrwxrwx 1 root root   9 Dec  6 20:18 usb-Logitech_USB_Optical_Mouse-event-mouse -> ../event0
lrwxrwxrwx 1 root root   9 Dec  6 20:18 usb-Logitech_USB_Optical_Mouse-mouse -> ../mouse0
```

For example, let us use `usb-0c45_USB_Keyboard-event-kbd` device as keyboard,
and `usb-A4TECH_USB_Device-if01-event-mouse` as mouse. (If your physical devices
export many event devices, like here, you may need to try each one of them;
usable device IDs usually ends in `-event-kbd` and `-event-mouse`.) So we should
specify `-k event1 -m event7` when starting `hid2bluetooth`.

### Output (bluetooth) device

Run `bluetoothctl list` to show device MAC addresses and names.
It will produce the following output:

```
Controller DC:A6:32:77:4B:71 daiyousei [default]
```

Run `hcitool dev` to show device MAC addresses and device names.
It will produce the following output:

```
hcitool dev
Devices:
        hci0    DC:A6:32:77:4B:71
```
Once you determined which one to use, specify it like `-b hci0`
when starting `hid2bluetooth`. By the way, default is `hci0`, so if you
have a single bluetooth device, it is most likely `hci0`, and you may omit
`-b` argument at all in such case.

# Operation

## Requirements

Script is written in Python 3.

You should run `hid2bluetooth` as root.

Additionally, you should add `Class = 0x0005c0` into `[General]` section
of `/etc/bluetooth/main.conf` (or, more correctly, set this class
to the chosen device by a command like `hciconfig hci0 class 0x5c0`), and
restart bluetooth service by a command like `systemctl restart bluetooth`.

Python dependencies required to run: `python3-dbus`, `python3-evdev`.

## Running

Just run `hid2bluetooth` with `-b`, `-k`, and `-d` arguments with corresponding
values.

Once started, it will start up threads that consume device inputs, and then
start pushing collected events int bluetooth.

You'll see output like this:
```
Using bluetooth:     hci0
Device MAC Address:  DC:A6:32:77:4B:71
Device Name:         daiyousei
Using keyboard:      event0
Device Name:         Logitech USB Optical Mouse
Using mouse:         event1
Device Name:         USB Keyboard
Started collecting keyboard events
Started collecting mouse events
Waiting for connections...
```
Then you can pair and connect your mobile device (or what you want to use
as a thing bluetooth keyboard and mouse are connected to), and then you
immediately get all inputs you produce with selected keyboard and mouse
on your mobile device.

If your device is not automatically connected or not paired, you may have
need to use `bluetoothctl` to pair with it.

You may terminate execution with Ctrl+C.

## Notes

Note that events from grabbed devices are ignored by OS since they are
been forwarded to bluetooth. You may have need to disable grabbing
by using `-g` argument, but then think about configuring Xorg or any
other display server accordingly, to avoid accidental unwanted actions
caused by input events from these devices while you use them to interact
with your mobile device.
