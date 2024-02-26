# HACS - MIoT Auto - Customize.yaml
This guide written in Chinese, target for Hong Kong newbies

Changlog
20240226 - Initial github version

## 先講廢話
其實只係怕自己唔記得, 加上龍哥又有提過, 咪寫低佢囉。
MIoT Auto 係 Home Assistant Community Store (HACS) 入面既一個由第三方提供既 Integration, 俾你 Link up 米家 (MiHome), 咁你可以在 HA 使用已上咗米家既設備, 我自己常用既用途
- 入咗 HA 後 Expose 番出去 HomeKit, 令本來無 HomeKit 支持既設備都在 HomeKit 度用到
- 因為可以同時用不同 Mi Account / 同一 Account 但不同區 Link Up MiHome, 嘿嘿嘿... 在同一個地方可以亙相 Access (當然對於 "硬鎖區" 既設備都係幫唔到手）

呢篇文係針對用 MioT Auto 加入設備後, Default Dashboard 沒有顯示出來既 Sensor / 功能, 但 https://home.miot-spec.com 上話有的. 最常見就係抽濕機水箱水滿, 或者水機無水要入水. MiHome 有提示但 HA by default 又無。
<br/>

## 點樣攪?

官方參考可以去呢度睇
https://github.com/al-one/hass-xiaomi-miot?tab=readme-ov-file#customize-entity<br/>

0. 強烈建議使用 Studio Code Server 呢個 Add-on. 咁你喺 Browser 直接修改 File 就 Apply Change, 日後非常方便。如果未裝, 你可以在 Settings -> Add-Ons -> Home Assistant Community Add-ons 搵到 

<br/>

1. 在 HA 入面既 /config/configuration.yaml 入面, 分別在 section homeassistant: 加入 "customize: !include customize.yaml", 同 加多個 xiaomi_miot: !include xiaomi_miot_auto.yaml
<br/>
   請參考附圖, 留意空格。  咁你就define左兩個 yaml
   
![addons](https://github.com/jomud9/homeassistant_howto/blob/5ba732c970ff9d0046d32548e4ba978122821bcc/images/miot_auto_cust01.jpg)

<br/>

2. 建立 /config/customize.yaml 同 /config/xiaomi_miot_auto.yaml. 為左方便大家, 你可以 Copy and Paste 下面幾行, 再自己改。

   唔駛擔心, 只係無打錯空格係唔會有乜大影響, 而有乜野改動 Apply 時請請在 Developer Tools 度, YAML Tag -> Check and Restart -> Check Configuration. 有野錯都幫到你。

/config/customize.yaml
```shell
#miot auto custom
humidifier.dmaker_22l_5d5d_dehumidifier:
  sensor_properties: "fault"

water_heater.chunmi_tsj13_866a_water_dispenser:
  sensor_properties: "fault"  
  
fan.zhimi_fb1_fcce_fan:
  select_properties: mode
  preset_modes: []


fan.huayi_fanwy_d39a_fan:
  select_properties: mode
  preset_modes: []
```

/config/xiaomi_miot_auto.yaml
```shell
translations:
  # Dictionary for specifying fan modes
  fan.mode:
    straight wind: "直吹模式"
    natural wind: "自然風"

device_customizes:
  miaomiaoce.airm.air01:
    miot_local: false # Force to read and write data in LAN (integrate by account)
    miot_cloud: true # Enable miot cloud for entity (read, write, action)

exclude_state_attributes:
  - miot_type
  - stream_address
  - motion_video_latest
```
<br/>

>- 其實攪番個 Entity 出來, 多數都係用 customize.yaml. 但有時想自定果D字, 係要經 xiaomi_miot_auto.yaml 
>- 唔知點解, Device 有D Status by default 係唔會出, 最常見就係抽濕機水箱水滿, 或者水機無水要入水
>- 詳情要參考番官網 Reference
> 可以用 # 間開, 或者係行頭度加 # 去 Remark/Disable 果行

<br />

3. 去 Developer Tools 度, YAML Tag -> Check Configuration -> Restart HA -> Restart Home Assistant (Full Restart)
   
<br/>


## 要點用?
其實都係睇下官網 Forum, 或者去 Telegram group果度抄下人地功課咁. 但原理係。
- 先睇下 https://home.miot-spec.com 果度, API 有冇果個 Data
- 再去 Developer Tools -> States Tab 度, Based on Entity 搵下有關既 Entity 入面有冇 miot spec 果度既 status
![addons](https://github.com/jomud9/homeassistant_howto/blob/b031638351bb9e8fd12ede10f7bfe5143a53f807/images/miot_auto_cust02.jpg)
- 在 /config/customize.yaml 入面加番有關既 Entity 同 有關既 Defination.

例如, 我呢架水機 https://home.miot-spec.com/s/chunmi.ysj.tsj13 在 Miot Auto 加入 HA 後, 係見到 Status (Idle) 但見不到 Fault. 
於是就係用番上面咁講, 見到 water_heater.chunmi_tsj13_866a_water_dispenser 見到有 "water_dispenser.fault" 呢個 Attributes, 就在 customize.yaml 做野了 
![addons](https://github.com/jomud9/homeassistant_howto/blob/b7b5463d965c6fe62122d61bedf41701d6ecf32b/images/miot_auto_cust03.jpg)

### 希望幫到你. ^__^
