# homeassistant_howto

Just collective notes for setting up my HA related stuffs

1. Note for Smart Switch and some tips before decoration work starts
   https://github.com/jomud9/homeassistant_howto/blob/main/groundwork_tips.md

2. How to update Sonoff Zigbee 3.0 (ZBDongle-P) USB firmware using a Mac
   https://vandalen.dev/post/how-to-update-sonoff-zigbee-3-0-zbdongle-p-usb-firmware-using-a-mac/
   - Install Python requirements
      > pip3 install pyserial intelhex
   - Download and extract cc2538-bsl
      > mkdir cc2538-bsl <br />
      > cd cc2538-bsl <br />
      > curl -sSL https://github.com/JelmerT/cc2538-bsl/archive/refs/heads/master.tar.gz | tar xz --strip 1
   - Locate USB Port
      > ls /dev/tty* | grep usb
   - Command line
      > python3 cc2538-bsl.py -ewv -p /dev/tty.usbserial-0001 --bootloader-sonoff-usb ./CC1352P2_CC2652P_launchpad_coordinator_20220219.hex

3. About Aqara Z1 Smart Wall Switch and pair with zigbee2mqtt 
   https://github.com/jomud9/homeassistant_howto/blob/main/onboarding_aqara_z1_switch.md
