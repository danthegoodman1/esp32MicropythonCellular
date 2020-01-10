# ESP32 TTGO T-Call SIM800L 2G Cellular MicroPython <!-- omit in toc -->

- [Installing MicroPython (LoBo) on ESP32](#installing-micropython-lobo-on-esp32)
    - [Clone MicroPython repo](#clone-micropython-repo)
    - [Build the firmware](#build-the-firmware)
    - [Configure GSM](#configure-gsm)
    - [Build and Flash Device](#build-and-flash-device)
- [Setup](#setup)
    - [Connect to repl](#connect-to-repl)
    - [Connecting to WiFi](#connecting-to-wifi)
    - [Creating a WiFi connect function](#creating-a-wifi-connect-function)
- [Internet](#internet)
- [Texting (send and receive)](#texting-send-and-receive)
- [Running files](#running-files)
- [Documentation and Links](#documentation-and-links)

## Installing MicroPython (LoBo) on ESP32

In order to use the GSM network on MicroPython, you need to [use a specific version of MicroPython by loboris](https://github.com/loboris/MicroPython_ESP32_psRAM_LoBo). This distribution was designed with the ESP32 in mind, and many of the example use the same pinout as the TTGO T-Call.

In order to install, we have to download and build the software ourselves, then flash the device.

**Requirements:**
- esptool
- pyserial `pip install pyserial`

#### Clone MicroPython repo

`git clone --depth 1 https://github.com/loboris/MicroPython_ESP32_psRAM_LoBo.git`

#### Build the firmware

`cd` into the folder, and find the `BUILD.sh` file. Then run `./Build.sh menuconfig`

#### Configure GSM

We have to enable GSM in our build, since normally the build tries to be a slim as possible by omitting features by default.

First, enter the `Serial Flasher Config` menu and make sure the right serial port is selected. If not, change it to the correct port.

Second, go to `MicroPython → Modules` and select `Use GSM Module`

Third, `Component config → LWIP → Enable PPP support → Enable PAP support.`

Fourth, `Component config → LWIP → Enable PPP support.`

Then you can exit, build, and flash the device.

#### Build and Flash Device

To build, run the command `./BUILD.sh`. This will build the firmware.

Once that completes, you have to run the command `./BUILD.sh flash`

_See more options [here](https://github.com/loboris/MicroPython_ESP32_psRAM_LoBo/wiki/build) for BUILD.sh_

Now we have MicroPython installed, and can use other tools for interacting just like any other MicroPython install (ampy for files, screen for repl sessions).

## Setup

The first thing we are going to want to do is setup a file that will have the packages we will need for this tutorial. By creating a file, we can upload and run it using the `ampy` utility. If you name the file `boot.py`, the file will run on boot. Anything else will have to be invoked either in the boot.py file, or in the repl.

Let's create a file called `test.py`, and start it with the following:

```py
import machine, time, sys
import gsm
import urequests



def main():
    GSM_APN  = 'wholesale' # Your APN
    GSM_USER = '' # Your User
    GSM_PASS = '' # Your Pass

    # Power on the GSM module

    GSM_PWR = machine.Pin(4, machine.Pin.OUT)
    GSM_RST = machine.Pin(5, machine.Pin.OUT)
    GSM_MODEM_PWR = machine.Pin(23, machine.Pin.OUT)

    GSM_PWR.value(0)
    GSM_RST.value(1)
    GSM_MODEM_PWR.value(1)
    print("Here we go!")

    # gsm.debug(True)  # Uncomment this to see more logs, investigate issues, etc.

    gsm.start(tx=27, rx=26, apn=GSM_APN, user=GSM_USER, password=GSM_PASS)

    sys.stdout.write('Waiting for AT command response...')
    for retry in range(30):
        if gsm.atcmd('AT'):
            break
        else:
            sys.stdout.write('.')
            time.sleep_ms(5000)
    else:
        raise Exception("Modem not responding!")
    print()
```

This will ensure that our cellular functionality is ready to go before we continue. We will add to this function with the various features below. Once we get this setup, we need to get into a repl and connect to the internet so we can install more packages. You could either use cellular to connect to the internet, but since packages are large let's just use wifi:

#### Connect to repl

run the following to enter the repl, changing your port as necessary
`screen /dev/cu.SLAB_USBtoUART 115200`

Then, we need to connect to the wifi:
. Once in the repl (hit enter a few times to ensure you are in python, or type `help()`) run the following commands:

#### Connecting to WiFi

```py
import network
sta_if = network.WLAN(network.STA_IF)
sta_if.active(True)
sta_if.scan() # Only necessary if you want to scan for networks, this can be valuable in showing the real network name (sometimes the encoding is insane, especially for my google home wifi network)
sta_if.connect('SSID', 'PASS')
```

Once connected, run the following commands to install `urequests`
```py
import upip
upip.install("urequests")
```

#### Creating a WiFi connect function

This will be useful for later. Let's create a connect function:

```py
def connectWifi():
    import network
    sta_if = network.WLAN(network.STA_IF)
    if not sta_if.isconnected():
        print('connecting to network...')
        sta_if.active(True)
        sta_if.connect('<YOUR WIFI SSID>', '<YOUR WIFI PASS>')
        while not sta_if.isconnected():
            pass
    print('network config:', sta_if.ifconfig())
```

And while we are at it, let's make a disconnect function as well:

```py
def disconnectWifi():
    import network
    sta_if = network.WLAN(network.STA_IF)
    sta_if.disconnect()
```

## Internet

Using the internet through cellular is very powerful feature. In order to connect to the internet (vs. normal cellular polling), we need to run the `gsm.connect()` method. Add this to our `main()` function:

```py
gsm.connect()

while gsm.status()[0] != 1:
    pass

print('IP:', gsm.ifconfig()[0]) # Local ip

print("Making get request...")
response = urequests.get("https://bot.whatismyipaddress.com")
print(response.text) # Internet ip
```

You could also use the socket library:

```py
import socket
addr_info = socket.getaddrinfo("towel.blinkenlights.nl", 23)
addr = addr_info[0][-1]
s = socket.socket()
s.connect(addr)

while True:
    data = s.recv(500)
    print(str(data, 'utf8'), end='')
```

_This will play a star wars movie in ascii_

## Texting (send and receive)

One of the awesome features is being able to test with python. In order to use texting, we have to disable the gsm modem if we were previously using it:

```py
gsm.disconnect()
```

Then we can begin to use the gsm methods:

```py
gsm.sendSMS("+1XXXyyyuuuu", "Hello from ESP32!")
```

You can also text using different formats for the phone number, ex:

`+1AAAdddeeee`

or just area code:

`AAAdddeeee`

The library does have the method for setting up a callback when you get a message, but I haven't been able to get it to properly work. So instead, as part of some main loop you should run the following function to check incoming texts:

```py
print("Checking messages...")
messages = gsm.checkSMS()
print(messages)

for index in messages:
    msg = gsm.readSMS(index, True)
    if (msg):
        print("\n\nMessage {}:".format(index))
        print("\nRaw:")
        print(msg)
        print("End Raw\n")
        print("New message from {}:".format(msg[2]))
        print(msg[6])
        print("\n")
```

## Running files

First, put the file onto the device:

`ampy -p /dev/tty.SLAB_USBtoUART put test.py`

`ampy` is a great tool for putting and getting files, however I don't like it for running them based on how it times out. To run the file we made, run the following commands in the repl:

```py
import test
test.main()
```

## Documentation and Links

ampy: https://github.com/scientifichackers/ampy

gsm documentation: https://github.com/loboris/MicroPython_ESP32_psRAM_LoBo/wiki/gsm
