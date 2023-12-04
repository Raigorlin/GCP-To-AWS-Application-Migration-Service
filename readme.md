# GCP (Compute Engine) 遷移 AWS (EC2)

首先這篇議題上, 我們會需要用到 AWS Application Migration Service (MGN) 來實現雲對雲的轉移

遷移架構如下：

![Alt Text](/Screenshots/AWS-MGN-Network-Architecture-Modernization.png)

## 前置作業

- [初步建置以及IAM權限設定](#初步建置以及iam權限設定)
- [VPC建置以及支援地區](#vpc建置以及支援地區)
- [EC2 模板安全規則設定](#ec2-模板安全規則設定)
- [MGN 設定](#mgn-application-migration-service-設定)

## 部署流程

1. [安裝 Agent (GCP Compute Engine)](#安裝-agent-gcp-compute-engine)
2. [等待初步同步](#等待初步同步)
3. [啟動測試EC2](#啟動測試ec2)
4. [確認啟動規格是否符合需求](#確認啟動規格是否符合需求)
5. [標記測試完畢](#標記測試完畢)
6. 啟動正式切換EC2
7. 停止遷移機器相關服務(請勿關機)
8. 最終確認正式切換EC2
9. 將MGN上的已經機器封存


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
#### 啟動測試EC2
---
#### 確認啟動規格是否符合需求
---
#### 標記測試完畢
---
#### 啟動正式切換EC2
---
#### 停止遷移機器相關服務(請勿關機)
---
#### 最終確認正式切換EC2
---
#### 將MGN上的已經機器封存
---

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