# Expose Home Assistant Entity to HomeKit via yaml 
This guide written in Chinese, target for Hong Kong newbies

### 先講廢話
其實點解有 Integration 唔用, 要用 yaml 去設定? ~~主要因為 HA 廢 :D~~
- Expose 出去既 Entities 如果其中一隻有事, 佢會炒左成個 Subsystem, 睇 Log 唔容易搵到
- 當你有好多 Entities 時, 你在 HomeKit Integration 要 Include 個 drop down list, 可以長蛇春咁長
- 你用 Integration expose 出來後, 你要 Add 入 HomeKit 時, **每一次重新加佢都會問你果隻Entity叫做乜野名**, 如果你想用中文或指定既 HomeKit 名稱, 你要自行在 Home.app 度輸入, 而用 yaml 係可以在 expose 果下已 define homekit既 device名

<br />
## 點攪? <br/>
官方參考可以去呢度睇
https://www.home-assistant.io/integrations/homekit/

<br/>

0. 強調建議先安裝 Studio Code Server 呢個 Add-on卜. 佢可以令你在 Browser 直接修改 File, 日後非常方便
> 如果未裝, 你可以在 Settings -> Add-Ons -> Home Assistant Community Add-ons 搵到 
![addons](https://github.com/jomud9/homeassistant_howto/blob/2fabe48e35cf15868b732a85c16f21bcc558b43b/images/homekit_yaml_01.jpg)

1. 在 HA 入面既 /config/configuration.yaml 入面, 加入下面既 definition (唔識可以參考上面張圖, 搵個空位加落去就係)
> homekit: !include homekit.yaml

2. 建立 /config/homekit.yaml, 為左方便大家、你可以 Copy and Paste 下面幾行, 再自己改
```shell
- name: HA Bridge
  mode: bridge
  filter:
    include_entities:
      #大廳
      - light.spotlights
      - switch.switchz1_livingroom_button_3
      - fan.zhimi_fb1_fcce_fan

      #書房
      - light.aqara_mainlight
      - sensor.mitemppro_humidity
      - sensor.mitemppro_temperature

  entity_config:
    #大廳
    light.spotlights:
      name: 射燈
    switch.switchz1_livingroom_button_3:
      name: 窗邊燈帶
    fan.zhimi_fb1_fcce_fan:
      name: 風扇仔
 
    #書房
    light.aqara_mainlight:
      name: 圓燈
    sensor.mitemppro_humidity:
      name: 書房濕度
    sensor.mitemppro_temperature:
      name: 書房溫度
      linked_battery_sensor: sensor.mitemppro_battery
      low_battery_threshold: 30
'''
> yaml 分三部份, Bridge definition, include entity 同 entity config. 
> 要 expose 既 entity 你可以在 Settings -> Devices and Services -> Entity Tab 度搵清楚先, 試清楚佢係唔係正常在HA用到先放落個 yaml 度
> entity_config 非必要, 但就係可以幫你 Assign HomeKit 名稱以及有關聯既 sensor information. 詳情要參考番官網 Reference

3. 去 Developer Tools 度, Check Configuration -> Restart HA -> Quick Reload

 或 

單純 Restart HomeKit YAML, 約10－15秒後 Home.app 就會更新 或者 要Force Quit 一次就睇到更新既 Entity
