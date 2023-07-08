## My journey with Aqara Z1 Smart Wall Switch

- 路人 JM9, Jun 2023
<br />

###### Changlog
###### 20230708: Draft for related file to allow z1 to be added into z2m

<br />
<br />


## Basic Information
First heard from https://homekitnews.com/2023/05/23/aqara-reveal-new-strip-ceiling-light-curtain-motor-and-switches at around May 2023, and I received mine (2 gang) at around Jun 21 thru Taobao.
![z1_inthebox](https://github.com/jomud9/homeassistant_howto/assets/130078583/961e28bf-ff91-4f53-8518-a1c74aa97f49)

Official information from Aqara web (in Chinese, as it releases on China Market at the moment
https://www.aqara.com/Canon_Smart_Wall_Switch_Z1_overview

<br />
<br />

Size comparsion with Aqara D1 Switch unit (which I am going to replace with a Z1) 
- from spec, Z1 is 86 x 86 x 3
![z1_vs_d1_01](https://github.com/jomud9/homeassistant_howto/assets/130078583/c35ff653-2213-4db0-8b51-1ee2299f0e4b) ![z1_vs_d1_02](https://github.com/jomud9/homeassistant_howto/assets/130078583/c6ebf994-b01c-484c-987a-1032561fdc9d)
<br />
<br />

ASMR for z1 click sound :P
https://github.com/jomud9/homeassistant_howto/assets/130078583/9485963d-9fb3-452c-abc6-d7d491df07f9
<br />
<br />

## Using Z1 wall switch with Zigbee2mqtt
Up to this moment (July 08, 2023), it is *NOT* perfect to use with z2m due to following reasons 
- action: ONLY single are supported after decoupled (no double, triple, hold). no multiple button pressed together (no both, all). (These are also *NOT AVAILABLE* in Aqara Home)
- flip_indicator_light(), power_outage_memory() and click mode() not working. ?? New Hardware New address?? (Aqara Home have these setting)


Latest external converter file available:
- 2-gang https://github.com/Koenkk/zigbee2mqtt/issues/18082
- 3-gang https://github.com/Koenkk/zigbee2mqtt/issues/18135

### Installation Step
1. Stop z2m
2. put related converter file ![editorview](https://github.com/jomud9/homeassistant_howto/blob/6c6516224cf0db91af668bd7a082920f0be3cd32/images/z1/z1_add_to_z2m.jpg) under /config/zigbee2mqtt/
3. Add the following section at /config/zigbee2mqtt/configuration.yaml
>`external_converters:` <br />
> `- ZNQBKG39LM.js`<br />
> `- ZNQBKG40LM.js`
  - This example are using two converter files
4. Re-start z2m. Check logs to see if z2m start successfully
5. Add z1 device as normal


Reference: https://www.zigbee2mqtt.io/advanced/support-new-devices/01_support_new_devices.html
