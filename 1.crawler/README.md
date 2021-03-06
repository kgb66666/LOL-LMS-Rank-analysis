## 前言 ##
首先把爬蟲(crawler)放在第一順位來講解，並不是代表研究的順序，而是整體來看  
爬蟲的部分可以快速帶過，所以把mongodb資料庫寫入的部分就變得沒什麼特殊，但爬蟲調適的耗時也不輸其他項目的。

## 開發工具 ##
<ul>
	<li>語言:Python3.6</li>
	<li>資料庫:mongodb</li>
	<li>資料庫空間:mlab、AWS</li>
	<li>額外插件:datetime、pymongo、requests、json</li>
	<li>爬蟲:Beautifulsoup4(一開始會使用，後來發現api獲得的json不必用到)</li>
	<li>Linux指令:nohup、crontab、kill</li>
</ul>

## 編寫多進程 ##

這邊運用Process來執行多"進"程    
所以有一個主程式py檔來設定到排程上，
用for 迴圈可以執行多個Process,這邊下了11個進程來呼叫py檔
start.py
```python
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

from multiprocessing import Process
import os
import time
def StartSh(path):
    os.system(path)

if __name__ == '__main__':
        
        sh='python crawler01.py '
        li = [sh]

        for i in range(1,11):

            p = Process(target=StartSh, args=(li[0]+str(i),))

            p.start()
```

之前是將腳本丟到AWS上去設定排程跑，不過免費的主機效能很低且多開發現流量會爆表，  
大概摸索了3個月就不在AWS上執行了，因為初衷是以免費為前提練習的，為此花了700多NT。  

## 設定排程 ##

我使用了crontab 來進行排程執行我要的腳本，這邊遇到了很多問題:  
<li>1.運行python start.py 後，關掉終端程序就停了!</li>
解決辦法使用 nohup運行python，就可以在後端運行python。  
<li>2.nohup 運行時，不支援utf-8。</li>
可以寫一個py檔來print(sys.stdout.encoding) ,  
可以在下圖看到用nohup python 執行會出現 None , 一般環境下執行會顯示utf-8 ,  
所以我們要Linux下指令，告訴Linux 環境 nohup 的python 預設需要使用utf-8,  
"export PYTHONIOENCODING=utf-8" ,按下Enter送出後，再次使用nohup 腳本就可以看到顯示utf-8。  

![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/nohup.jpg)
<li>3.運行nohup python start.py 後，有些進程不會自己離開，會佔用掉記憶體,最後AWS整個當掉。</li>
因為是免費的AWS所以memory 大小有限，所以crontab 也設定了一個排程用來移除殘存的進程。  
crontab */20 * * * * pkill -9 python，每20分鐘就刪除python所有進程。  

<li>4.crontab 	排程設定如果時間相同時，一個是運行python 一個是pkill python時，會隔次時間才開啟。</li>


## 爬蟲部分 ##
1.圖1是資料庫的部分  
1aws 代表 第一個進程，2aws 代表第二個進程，以此類推。  
字典(dict)裡面的  
start 是每次進程for迴圈裡面的第一個要爬的遊戲編號，  
here 是 當前運行時爬到哪個遊戲編號了，如果中間碰到程序問題還可以記錄還有多少沒爬完。  
edate 是當前爬到的遊戲時間  
end 是這個進程最後結尾的遊戲編號，也就是迴圈最後會到這個遊戲編號後停止。
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/07-1.jpg)  
2. 圖2是爬蟲之前會先去讀取資料庫裡面(1～11)個進程記錄，把here和end的遊戲編號都抓出來，  
之後透過max(list)的方式查看最新的end最後遊戲編號是多少，這裡查看到是1515871876，下方會藉由這個數字接著繼續爬蟲。  
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/07-2.jpg)
3.圖3 可以看到 將1515871876帶入下方function，這邊設定範圍爬取範圍10，也就是1515871876～1515871886，  
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/cr-1.jpg)  

## Json解析 
然後到官方的api上查詢遊戲資料，格式是Json，建議使用firefox瀏覽器開啟:  
https://acs-garena.leagueoflegends.com/v1/stats/game/TW/1515871876/timeline    
記錄著遊戲的每分鐘發生的事情，如購買販賣道具(ITEM_PURCHASED)、技能升級(SKILL_LEVEL_UP)、技能進化(EVOLVE)、擊殺英雄(CHAMPION_KILL)在"type"來分類。
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/cr-3.jpg)  
https://acs-garena.leagueoflegends.com/v1/stats/game/TW/1515871876  
結尾沒有timeline的是該場整體數據，後面戰績顯示和爬蟲都是依賴這個api網址。  

![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/cr-2.jpg)  
可以看到api返回了兩個json，這邊看起來很恐怖對吧~  
其實對照一下戰績後就可以得知一些資訊，例如＂queueId＂:430　是代表　'一般對戰'、"teams.0.teamId" :100 代表藍方陣營，200 代表紅方陣營。

## 遇到時間格式轉換 
gameDuration:1736 是該場遊戲時間長度，格式轉化後可以獲得時分秒。  
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/cr-4.jpg)  
gameCreation:1527272140991,這是一個時間戳(timestamp)格式,不過時間戳格式其實是10位數，所以只要取前10位數就行了，  

```python
datetime.datetime.fromtimestamp(int('1527272140')).strftime("%Y-%m-%d %H:%M:%S")
```
直接用timeago格式化可以得到時間。(timeago需要另外安裝插件)  
![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/cr-5.jpg)  

## Mongodb 寫入 
接下來是資料寫入Mongo的部分  
因為爬蟲有使用到一些mongo用法,所以來稍微記錄一下。  

在寫入資料之前會先查詢該英雄是否在資料庫內，如果找不到值會以None值回傳  
```python
version = '8.7'
champ_db_data = db[version].find_one({'_id': champid})

if champ_db_data == None:  # 如果名稱為8.7資料庫沒有該英雄資料，新增該英雄object到資料庫
    champ_db = db[version]
```
如果上面未找到相對應的'_id'這樣下面的update更新語法就會失敗，所以我們要用到insert新增資料表  
如果沒有db[version]這個資料庫，用insert也會一起順便建立db_  
update有set指令和inc指令，一個是複寫內容、一個是數值內容幫你加1，



```python
champ_db.insert({'_id': champid, 'version': version}) #寫入一筆資料，即使沒有db 也會順帶創出來

champ_db.update({'_id': champid}, {"$set": {'game_static': di}}) #update 如果用了"$set" 是整個複寫"game_static"內的內容

champ_db.update({'_id': champid}, {"$inc": {lll: l}}) #update 用 "$inc" 是用來將裡面的數值+1
```

### 以下為set例子:  
覆寫內容。這邊是將爬到的遊戲時間記錄到資料庫，所以更新"1aws"裡面的edate內容。  

![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/set.png)  

### 以下為inc例子:  
inc可以用在累加勝敗場次或是運算用，這邊是遊戲中使用id為154的英雄，如果該場勝利就在勝利欄位+1。

![](https://raw.githubusercontent.com/kenson2998/LOL-TW-Rank-analysis/master/1.crawler/img/inc.png)  
