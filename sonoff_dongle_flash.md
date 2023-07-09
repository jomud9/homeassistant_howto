## How to update Sonoff Zigbee 3.0 (ZBDongle-P) USB firmware using a Mac
It is via python to flash related firmware without open case
   https://vandalen.dev/post/how-to-update-sonoff-zigbee-3-0-zbdongle-p-usb-firmware-using-a-mac/
   - Install Python requirements
      ```shell
      pip3 install pyserial intelhex
      ```
   - Download and extract cc2538-bsl
      ```shell
      mkdir cc2538-bsl
      cd cc2538-bsl
      curl -sSL https://github.com/JelmerT/cc2538-bsl/archive/refs/heads/master.tar.gz | tar xz --strip 1
      ```
   - Locate USB Port
      ```shell
      ls /dev/tty* | grep usb
      ```
   - Command line
      ```shell
      python3 cc2538-bsl.py -ewv -p /dev/tty.usbserial-0001 --bootloader-sonoff-usb ./CC1352P2_CC2652P_launchpad_coordinator_20220219.hex
      ```
<br />

## How to update Sonoff Zigbee 3.0 (ZBDongle-E) Firmware from web flasher
It is originally from Smart Home Scene https://smarthomescene.com/guides/how-to-enable-thread-and-matter-support-on-sonoff-zbdongle-e/

>Since the Sonoff ZBDongle-E is based on the EFR32MG21 Multiprotocol SoC chip, it’s capable of both Zigbee and Thread communication thanks to the radio of the module. However, this device is sold solely as a Zigbee dongle by the company and there is no official Thread support at the moment. Whether Sonoff intends to push an update making the device act as a Thread Border Router is unknown at this moment.
>
>Luckily, you can flash the Sonoff ZBDongle-E version with a custom firmware and use it Home Assistant both as a Zigbee Coordinator as well as a Thread Border Router. The process is not difficult, but it does require some additional configuration and setup after you’ve flashed the device.

I did not have very good experience with newer NCP version but the flasher allow you to flash between NCP/RCP and custom firmware, it is a better means for you.

Please read the page carefully. 

![flasher](https://github.com/jomud9/homeassistant_howto/blob/2ecd3083ce6cee8ef2280150d9054bc3f9bbae7e/images/firmwareflasher.jpg)
