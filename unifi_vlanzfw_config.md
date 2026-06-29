# Ubiquiti UniFi VLAN and Zone Firewall Configration (in Chinese 廣東話)
This guide written in Chinese, target for Hong Kong newbies

Changlog
20260629 - Initial github version

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
1. Cody @ MacTelecom 既 Walkthrough: [UniFi Network 2026 Full Walkthrough Tutorial](https://www.youtube.com/watch?v=i3dsHaCVYAE) <br />
  你係可以一睇一路跟住做
1. Tom Lawrence 既 進階片: [UniFi Firewall Defaults Are Too Open: Here’s How to Lock Them Down](https://www.youtube.com/watch?v=EGls6JvdaNc) <br />
  佢講既內容如果你有其他器材既經驗, 會易上手少少
<br/>


## FAQ
1) 選購建議 (同本文好似唔太大關係但... 你買錯野就無得玩)
- 新手唔好以為 UniFi Dream Router 7 (UDR7) 好好，因為玩 UniFi 要諗 AP 走位, Router 同 AP 一體既 UDR7 限制放置位置。當然地方細唔會加 AP 好似無大問題, 但儲存用SD Card令佢想行UProtect做NVR又企咗係度, 兩面不是人
- 想平玩 UniFi AP, 可以咸魚買 AC-PRO 呢類前朝機王 (約 250-300 RMB) 玩住先。當然呢篇文提及既VLAN/Zone Bone Firewall就用唔著／用唔到了
- PoE Switch 非必要 (可以用 PoE Injector供電)
2) Default Zone 有指定用法, 彈性有限. 例如 Cody 條片教 Guest Network 放入 Hotspot Zone (及 Disable Guest Portal Page). 但如果你想俾用緊 Guest Network 用 Apple Airplay Screen Mirroring 或 HomePod 播歌, 就點加 Rules 都唔可以
3) UniFi Config File 可以 Download 作備份. 改 VLAN/Firewall Setting 前咪留個底，有事set錯野大不了咪 Reset Factory Default -> Restore Backup就可以無限復活
4) 如果你覺得資料有問題或者有野想問, 可以去 Telegram Group https://t.me/smarthomehk 問大家。也歡迎 Fork 或者 submit changes/issues

   

### 希望幫到你. ^__^
