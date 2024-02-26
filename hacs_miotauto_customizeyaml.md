# HACS - MIoT Auto - Customize.yaml
This guide written in Chinese, target for Hong Kong newbies

Changlog
20240226 - Initial github version

## 先講廢話
其實只係怕自己唔記得, 加上龍哥又有提過, 咪寫低佢囉
<br/>

## 點樣攪?

官方參考可以去呢度睇
https://github.com/al-one/hass-xiaomi-miot?tab=readme-ov-file#customize-entity<br/>

0. 強烈建議使用 Studio Code Server 呢個 Add-on. 咁你喺 Browser 直接修改 File 就 Apply Change, 日後非常方便
> 如果未裝, 你可以在 Settings -> Add-Ons -> Home Assistant Community Add-ons 搵到 
![addons](https://github.com/jomud9/homeassistant_howto/blob/2fabe48e35cf15868b732a85c16f21bcc558b43b/images/homekit_yaml_01.jpg)

<br/>

1. 在 HA 入面既 /config/configuration.yaml 入面, 加入下面既 definition (唔識可以參考上面張圖, 搵個空位加落去就係)
```shell
homekit: !include homekit.yaml
```
<br/>

2. 建立 /config/homekit.yaml. 為左方便大家, 你可以 Copy and Paste 下面幾行, 再自己改
```shell
- name: HA Bridge
  mode: bridge
  filter:
    include_entities:
      #大廳
      - input_boolean.ventmode1
      - switch.switchz1_livingroom_button_3
      - fan.zhimi_fb1_fcce_fan

      #書房
      - light.aqara_mainlight
      - sensor.mitemppro_humidity
      - sensor.mitemppro_temperature

  entity_config:
    #大廳
    input_boolean.ventmode1:
      name: 大廳通風
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
```
<br/>

>- yaml 簡單來講分為三部份, Bridge definition, include entity 同 entity config. 
>- 要 expose 既 entity 你可以在 Settings -> Devices and Services -> Entity Tab 度搵清楚先, 試清楚佢係唔係正常在HA用到先放落個 yaml 度
>- entity_config 非必要, 但就係可以幫你 Assign HomeKit 名稱以及有關聯既 sensor information. 詳情要參考番官網 Reference
> 可以用 # 間開, 或者係行頭度加 # 去 Remark/Disable 果行

<br />

3. 去 Developer Tools 度, Check Configuration -> Restart HA -> Quick Reload

   或 

   單純 Restart HomeKit YAML, 約10－15秒後 Home.app 就會更新 
   
   如果無反應, 就要在 iOS 果邊 Force Quit Home.app 一次，咁遁睇到更新既 Entity

<br/>

4. 在極端情況下, 發覺有問題要 troubleshoot, 你可以試試
    - 先 Check HA Logs file
    - Reboot Home Assistant 一次 <br/>
    ![reboot](https://github.com/jomud9/homeassistant_howto/blob/3d5a9ba7a9e362f0ddc8f89b2c4f3d2fa09eb05c/images/ha_reboot.jpg)
    - 在 Home.app 度 Remove HA expose 的 Bridge 再加番
    - 加 # 號 Disable 所有在 include entities 果 part 既 entity, 再去 Restart Homekit yaml, 睇下見到隻Bridge後, 再 add 番 HA expose 的 bridge. 再以同方式去一行一行咁移除 * 號

<br/>

## FAQ
1. 如果不到 150 Device, 請不要開 Multiple Instances. 一來食多左 Resource, 二來如果有 Bad Entity 炒 HomeKit Subsystem, 所有 HomeKit Bridge 都係會炒晒
2. HomeKit 對某D Entity 有限制, 未必所有 HA 既 Entity 都可以 Expose 去到 HomeKit. 詳見 Reference
3. 入左 HA 既 WebCam, 理論上都可以用呢個方法入 HomeKit, 但想效果好, 不如用 Scrypted. https://github.com/koush/scrypted/wiki/Installation:-Home-Assistant-OS
4. 想用 HomeKit 叫番 HA 行 Automation, 最簡單既方法, 係
   - 用 HA Helper, create 個 input_boolean
     https://www.home-assistant.io/integrations/input_boolean/
   - expose 佢去 HomeKit
   - 在你的 HA Automation / Nodered 去 check 呢個 entity as trigger
   

### 希望幫到你. ^__^
