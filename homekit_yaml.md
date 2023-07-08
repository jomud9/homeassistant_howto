# Expose Home Assistant Entity to HomeKit via yaml 
This guide written in Chinese, target for Hong Kong newbies

### 先講廢話
其實點解有 Integration 唔用, 要用 yaml 去設定? ~~主要因為 HA 廢 :D~~
- Expose 出去既 Entities 如果其中一隻有事, 佢會炒左成個 Subsystem, 睇 Log 唔容易搵到
- 當你有好多 Entities 時, 你在 HomeKit Integration 要 Include 個 drop down list, 可以長蛇春咁長
- 你用 Integration expose 出來後, 你要 Add 入 HomeKit 時, **每一次重新加佢都會問你果隻Entity叫做乜野名**, 如果你想用中文或指定既 HomeKit 名稱, 你要自行在 Home.app 度輸入, 而用 yaml 係可以在 expose 果下已 define homekit既 device名

<br />
## 點攪?

0. 強調建議先安裝 Studio Code Server 呢個 Add-on (在 Settings -> Add-Ons -> Home Assistant Community Add-ons ) 
!(/images/homekit_yaml_01.jpg)
