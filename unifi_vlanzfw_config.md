# Ubiquiti UniFi VLAN and Zone Firewall Configration (in Chinese 廣東話)
This guide written in Chinese, target for Hong Kong newbies

Changlog
202360629 - Initial github version

## 又先講廢話
~~垃圾牌~~ Ubiquiti UniFi 香港因為有新代理，所以2026年係香港市場都常見左. 作為一個由2017年開始用呢牌子既用戶，到 2024年 拎左 UniFi Full Stack Professional (UFSP) 同 UniFi Wireless Admin 官方認證, 我一路屌一路用
#### 屌佢既有
- 近年 Firmware 不穩, 搵正常用戶當 Beta User, 你 Auto Update 既下場就係斷線同上唔到網
- 再屌就係越賣越貴. UniFi G3 Instant USD29 既好年代不再
- 三屌就係以前香港無代理要買機上美國官網買要自己俾運費, 但官網會 Cut 單 (但當然有方法), 但買左出事要 RMA 寄番美國的話, 流程煩過西 (我當然都試過, 也不是一次)
- ~~四屌就係你買左第一件野, 就會自自然然買落去, 越買越多, 越玩越大~~
- 香港2024年開始有所謂行貨，到2026年有代理幫 U7系列既 AP去 OFCA拎 Label, 全線產品香港官方比美方定價約為 10 到 12算 (USD199 賣 2000 HKD), 呢層我唔會屌佢，因為你可以唔買
#### 重繼續用嘅原因
- GUI UX 雖然比以前複習左好Q多, 但你玩過除 OpenWRT 既其他野 如 pfSense, Mikrotik 等, 你會覺得佢叫易用
- 有套生態: Router, Switch, AP, Camera, NVR, SIP, Sensor....
- Home Assistant Friendly
- 叫做有牌子既 PoE Switch 或者 10GbE L2/L3 Switch, 除左 Mikrotik 到佢叫做最平
- 外國 Homelab 界好多人用, 所以教學 Youtube 片睇到你唔想睇
- 一路以來, Ubiquiti 既 Community Support Forum ~~又叫慘叫牆~~好多人幫手, 個隻小圈子 Support Group, 好似N年前既 Mac 社群個自助感覺

<br/>

## 唔想睇字?
我會推介你睇晒下面果兩條片
1. Cody @ MacTelecom 既 Walkthrough: [UniFi Network 2026 Full Walkthrough Tutorial](https://www.youtube.com/watch?v=i3dsHaCVYAE)
  你係可以一睇一路跟住做
1. Tom Lawrence 既 進階片: [UniFi Firewall Defaults Are Too Open: Here’s How to Lock Them Down](https://www.youtube.com/watch?v=EGls6JvdaNc)
  你係可以 
<br/>

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
