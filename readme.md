# GCP (Compute Engine) 遷移 AWS (EC2)

首先這篇議題上, 我們會需要用到 AWS Application Migration Service (MGN) 來實現雲對雲的轉移

遷移架構如下：

![Alt Text](/Screenshots/AWS-MGN-Network-Architecture-Modernization.png)

## 最佳做法 (官方推薦)

- 計畫
    1. 將預計要遷移的機器上安裝 AWS複製Agent
    2. 在遷移機器切換過程請勿重啟
    3. 請勿按封存或切斷連線到AWS在EC2上運行遷移機器並且確認完畢為止
- 測試
    1. 遷移機器在測試步完成後驟官方建議至少等待兩周來確認是有潛在問題或連線問題
    2. 請確保在切換遷移機器前做完整的測試
- 成功導入遷移機器
    1. 部署AWS複製Agent 到要遷移的機器上
    2. 確保複製狀態為healthy 
    3. 至少在正式切換前一個禮拜測試複製好的資料
    4. 紀錄任何發現的錯誤，EC2部屬設定錯誤以及確認AWS官方的相關限制

## 前置作業

- [初步建置以及IAM權限設定](#初步建置以及iam權限設定)
- [VPC建置以及支援地區](#vpc建置以及支援地區)
- [EC2 模板安全規則設定](#ec2-模板安全規則設定)
- [MGN 設定](#mgn-application-migration-service-設定)

## 部署流程

1. [安裝 Agent (GCP Compute Engine)](#安裝-agent-gcp-compute-engine)
2. [等待初步同步](#等待初步同步)
3. [啟動測試EC2](#啟動測試ec2)
5. [標記測試完畢](#標記測試完畢)
6. [啟動正式遷移機器至EC2](#啟動正式遷移機器至ec2)
7. 停止遷移機器相關服務(請勿關機)
8. 最終確認遷移完畢後的機器狀態
9. [將MGN上的已經遷移機器封存](#將mgn上的已經機器封存)


### 初步建置以及IAM權限設定

在第一次啟用這個服務時 AWS 會創建以下幾個預設 IAM 角色

- AWSServiceRoleForApplicationMigrationService 
- AWSApplicationMigrationReplicationServerRole
- AWSApplicationMigrationConversionServerRole
- AWSApplicationMigrationMGHRole
- AWSApplicationMigrationLaunchInstanceWithDrsRole
- AWSApplicationMigrationLaunchInstanceWithSsmRole
- AWSApplicationMigrationAgentRole

以及幾個預設的權限規則

- AWSApplicationMigrationFullAccess (MGN 相關的所有權限)
- AWSApplicationMigrationEC2Access  (MGN 用來創建/啟動 EC2測試機以及遷移機器權限)
- AWSApplicationMigrationSSMAccess  ([詳細介紹/設定參考這裡](https://docs.aws.amazon.com/zh_tw/systems-manager/latest/userguide/what-is-systems-manager.html))
- AWSApplicationMigrationReadOnlyAccess (用來檢視MGN 的設定)
- AWSApplicationMigrationAgentInstallationPolicy (用來安裝MGN Replication Agent的權限)
- AWSApplicationMigrationServiceEc2InstancePolicy (在AWS不同資源區EC2的遷移以及安裝MGN Replication Agent)


但這次我們這邊將套用 AWSApplicationMigrationAgentInstallationPolicy 用來安裝 Agent 

![Alt text](/Screenshots/image.png)

![Alt text](/Screenshots/MGN-IAM-01.png)

![Alt text](/Screenshots/MGN-IAM-02.png)

![Alt text](/Screenshots/MGN-IAM-03.png)

![Alt text](/Screenshots/MGN-IAM-04.png)

![Alt text](/Screenshots/MGN-IAM-05.png)

![Alt text](/Screenshots/MGN-IAM-06.png)

![Alt text](/Screenshots/MGN-IAM-07.png)

![Alt text](/Screenshots/MGN-IAM-08.png)

### VPC建置以及支援地區

支援地區請參考這邊[https://docs.aws.amazon.com/mgn/latest/ug/supported-regions.html](https://docs.aws.amazon.com/mgn/latest/ug/supported-regions.html)

這次我們這邊用日本東京地區 (ap-southeast-1) 來做示範

![Alt text](/Screenshots/MGN-VPC-01.png)

![Alt text](/Screenshots/MGN-VPC-02.png)

![Alt text](/Screenshots/MGN-VPC-03.png)


### EC2 模板安全規則設定

![Alt text](/Screenshots/MGN-EC2-01.png)

![Alt text](/Screenshots/MGN-EC2-02.png)

![Alt text](/Screenshots/MGN-EC2-03.png)

### MGN Application Migration Service 設定

![Alt text](/Screenshots/MGN-01.png)


#### 複製服務的 EC2 模板

> `[Note]`只要對象Agent 還在運行 模板服務的EC2 會一直在運轉

![Alt text](/Screenshots/MGN-02.png)

將網口改為先前建立好的VPC 子網段

![Alt text](/Screenshots/MGN-03.png)

####  EC2建置模板 (Test/Cutover)

這模板是用來套用複製好的instance 

![Alt text](/Screenshots/MGN-lunch-01.png)

####  EC2建置後續模板 (如果不需要可以忽略)

用來安裝AWS SSM agent至EC2 機器上主要用來監控

![Alt text](/Screenshots/MGN-postlunch-01.png)


### 部署流程

#### 安裝 Agent (GCP Compute Engine) 
---


#### 等待初步同步
---

遷移週期所顯示的狀態如下:
- Not ready : MGN服務正在複製遷移機器資料至AWS 
- Ready for testing : 複製完畢可以開始測試
- Test in progress : 正在部署相關服務以及機器
- Ready for cutover : 測試完畢可以部署正式要遷移的機器
- Cutover in progress : 正在部署正式遷移機器
- Cutover complete : 正式遷移完畢 (finalize Cutover Process)
- Disconnected : AWS複製Agent 已斷線會有兩種情形網路不穩定或者部署完畢 

#### 測試複製資料
---

在啟動複製的資料前你會需要確認EC2 啟動規格以及相關設定

> 如果你沒有選擇這個條件而是用自己的啟動模板可能要確認機器規格是否符合
![Alt text](/Screenshots/Deploy-test-02.png)

![Alt text](/Screenshots/Deploy-test-01.png)

啟動完畢後要確認是否需要外部IP來確認RDP 連線

![Alt text](/Screenshots/Deploy-test-03.png)

如果需要確認最後更新資料的時間

![Alt text](/Screenshots/deploy-lunch-history-02.png)

![Alt text](/Screenshots/deploy-lunch-history-01.png)

---
#### 標記測試完畢
---

![Alt text](/Screenshots/Deploy-test-ready-01.png)

---

#### 啟動正式遷移機器至EC2
---

步驟與上面相同 

`[Note]` 停止遷移機器上的所有服務請勿再進行寫入否則快照會無法同步但請勿關機AWS MGN需要確認AGENT 的存活狀態

![Alt text](/Screenshots/Deploy-lunch-cutover-01.png)

確認完畢後進行最終確認即可

![Alt text](/Screenshots/Deploy-lunch-cutover-02.png)

---
#### 將MGN上的已經機器封存
---

![Alt text](/Screenshots/Deploy-archieve-01.png)

可以在這邊到之前做的

![Alt text](/Screenshots/Deploy-archieve-02.png)

## TroubleShooting

如果跟MGN Stage網段無法溝通, 請檢察下列設定是否正確

1. 檢查Staging 網段的DHCP/ACL 規則是否正確
2. 確保能夠訪問外部DNS以及出口開放 443/tcp 
3. Stage 網段的路由可能設定錯誤

![Alt text](/Screenshots/MGN-VPC-01.png)

![Alt text](/Screenshots/MGN-VPC-troubleshoot-02.png)

![Alt text](/Screenshots/MGN-VPC-troubleshoot-03.png)

![Alt text](/Screenshots/MGN-VPC-troubleshoot-04.png)
---

如果你得到SSL 憑證錯誤

![Alt Text](/Screenshots/Windows-Migration-SSL_Trust-Failed.png)

### Steps 1
```powershell
# Check if 443 port is normal for connection to AWS application migration Service
Test-NetConnection mgn.{Region_ID}.amazonaws.com -Port 443
```
![Alt Text](/Screenshots/Test-NetConnection-Result.png)

### Steps 2

下載並且安裝在對應機器上

[Amazon Trust CA Verification and Download](https://www.amazontrust.com/repository/#)
![ALt Text](/Screenshots/Check-Browser-SSL.png)

## Referencing

https://repost.aws/knowledge-center/mgn-windows-replication-agent-install