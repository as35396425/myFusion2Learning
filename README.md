# photon network Fusion2 

## 名詞
- NetworkObject
- NetworkTransform
- StateAuthority 
    - client有inputAuthority去蒐集輸入，然後將輸入傳到有StateAuthority 的server or Host 處理後回傳給client端
- HostMode
    - 雲端+本地
- Share Mode
    - 雲端擁有stateAuthority
https://doc.photonengine.com/zh-tw/fusion/current/manual/
### HasStateAuthority 用來判斷local端
因為多人連線時會有多個相同物件，都掛載相同的腳本，但我們只需要操控本地端，所以使用hasStateAuthority來判斷本地玩家


---
- fusion2 玩家控制
    - 進行鍵盤輸入時，不要用FixedUpdateNetwork 來接收輸入
    - In Update do local input and change value of bool , 
    - 在FixedUpdateNetwork 判斷布林值


check whether the client side controlls an object
``` C# 
if (HasStateAuthority == false) 
```


- 玩家一律透過PlayerSpawner 生成 不須放置玩家物件到場景當中

``` C# 
public class PlayerSpawner : SimulationBehaviour, IPlayerJoined
{
    public GameObject PlayerPrefab;
    public Transform spawn ; 
    public void PlayerJoined(PlayerRef player)
    {   
        if (player == Runner.LocalPlayer)
        {
            Runner.Spawn(PlayerPrefab, spawn.position, Quaternion.identity);
        }
    }
}
```
### 屬性變更
利用Network屬性 或 RPC遠端呼叫函式
- RPC
    - 允許遠端呼叫函式 去改變對方的值
- Network
    - 數值被變更之後 被動觸發函式 
    - 用來同步數值 實際做法是直接改變network的數值 在將network拉到本地 
 



### 遇到的報錯

```
Fusion.CodeGen.ILWeaverBindings: (0,0): error  ---> Fusion.CodeGen.ILWeaverException: System.Void Health::DealDamageHealth(System.Single): name needs to start or end with the "Rpc" prefix or suffix
```
在需要用到RPC功能的函式名稱加上RPC的前綴或後綴 例如:
```C#
public void RPC_DealDamageHealth(float damage)
```

### photon Host 如何用到雲端

- Fusion支援UDP打孔，利用photon cloud 作為中繼Server以實現peer to peer 

- 在每次開啟host時，需要到雲端伺服器去驗證APP ID，但我們可以確定在運行時server與client直接連線

Runner deshboard
![image](https://hackmd.io/_uploads/H18BrzjnT.png)

以原始碼來看


---

這個 Scene Settings -> Peer Mode 會影響之後在 Play/Build (測試、建置) 時的執行模式, 目前有二種選擇, 如下:
- Single : 預設 單連線端
- Multiple : 多個連線端. (方便開發測試時使用) 執行多個 Fusion Runner;

有多個 Runner , 其中一個 Runner 做為 Server / Host, 其它做為 Client,


### Lag Compensation 參數用於調整命中判定hitbox

- Hitbox Buffer Length In Ms ->調整hitbox snapshot的儲存時間以及被蓋掉的時間(time of store snapshot be longer
- Hitbox Default Capacity 儲存 hitbox snap的數量
- Cached Static Colliders Size -> Lag Compensation 的

- Enqueue Incomplete Synchronous Spawns
	enable -> 將還沒完全生成的物件也一起同步生成，並enqueue
	disable -> throw exception 且 下一幀重新嘗試生成

- Invoke Render In Batch Mode unity的批次模式下啟用渲染
	Batch mode 用CLI方式開啟unity

- Network Id Is Object Name 
	產生物件名字包含NetworkID

- Hide Network Object Inactivity Guard
	Network Object Inactivity Guard為追蹤啟用前就被銷毀的物件訊息，NetworkObject的child gameobject
	開啟將在Hierarchy隱藏

### Simulation 負責遊戲網路同步集狀態更新
- Replication Features：Simulation支持不同的复制特性，例如：

    - None：預設 遊戲不超過幀數限制時選擇。
    - Scheduling：Network Object 無法被伺服器複製到客戶端，因為一個tick傳不完(因為超過傳輸限制 所以被culling過濾掉)，提高priority使該物件下一次優先被處理更新到客戶端。
    - Scheduling and Interest Management：出了上述功能，再增加客製化的過濾機制，可限制某Network object從伺服器更新，以及限制某NetworkBehaviour 從server更新
### Input Transfer Mode : 輸入用何種方式通過網路
- Redundancy : 冗餘輸入與壓縮，適合大部分遊戲
- latest : 直接傳送最新的狀態 適合VR 或大型輸入結構的遊戲
- Player Count 玩家數量
---
### shareMode專屬  所有設定都基於client tick rate 
- Client Tick Rate:8~256 範圍 客戶端的tick rate
- Client Send Rate: 比率參考上面，
	- 1/8: 需要超過240
	- 1/4 需要被4整除
	- 1/2
	- 1:1
- Server Tick Rate  server的更新速度
- Server Send Rate  不可超過server tick rate 
- Connection Timeout 逾時會斷開 沒收到更新的那端主動斷開
- Connecting Shutdown Time 狀態便shutdown 釋放資源的時間量
 
### Transfer Modes 選擇可靠資料傳輸的模式
- Client to Server能夠可靠的傳到伺服器
- Client to Client With Server Proxy 伺服器為代理，允許可靠的將客戶端傳到客戶端

### Host Migration 
- 自動更新到雲端 以及可設定延遲多久更新
### Network Conditions
- 模擬不同網路環境下的使用者 並測試 可調延遲或掉封包率

### Heap
- 調整記憶體管理策略，可使用預先分配給fusion的記憶體

### Weaver Settings 編譯比較low level 的網路程式碼
- Use Serialized Dictionary 勾選的話可以用Fusion.SerializeableDictionar 來存取網路屬性的初始值
- Null Checks for Networked Properties 檢查network behaviour 的關聯Network object
- Check Rpc Attribute Usage 檢查RPC是否被用在不支援的地方
- Check Networked Properties Being Empty 不正常使用NetworkedAttribute的話報錯

### Prefab 決定在不需要的時候卸載prefab
- Unload Prefab On Releasing Last Instance
    - 釋放時卸載prefab
- Unload Prefabs On Shutdown 
    - 關閉時卸載預製件
---
## Fusion state UI


### fusion grapgh 用於調整UI

可分為overlay 與 gameobject 
- gameobject 將UI作為物件形式放在場景
- overlay 平常的UI 呈現於螢幕上
可以調整大小以及顯示狀態項目，預設顯示以及詳細敘述為如下

- fusion state 生成UI組件用於監控狀態
- 預設狀態包含
    - 頻寬 in & out ( Byte ) 
    - 封包 in & out ( Packet )
    - RTT round trip time / ping 封包往返的時間
    - Resimulation 
        在本地端預測的速率
        需要網路狀態更新的行為都由模擬完成，模擬除了用作網路狀態更新外，還可以基於本地端資料去做預測
        狀態從StateAuthority更新
        -> network Object變為最新tick 
        -> 客戶端預測重新模擬  
        -> 更新到本地當前tick
    Referece:https://doc-api.photonengine.com/en/fusion/current/class_fusion_1_1_simulation.html    
    - forward step  
         resimulation後傳到local tick的流量
        from state authority receive snapshot 
        -> 傳到local prediction 
        -> resimulation forward to the local tick

bill board 需要camera元件，用於UI的相機跟蹤


