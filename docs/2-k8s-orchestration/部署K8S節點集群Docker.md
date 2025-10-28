# 部署K8S節點集群Docker

先將準備好的離線包使用 FileZilla 上傳到不同的 node 節點如下圖先配置站點

![](images/部署K8S節點集群Docker/2025-10-28-23-39-39-image.png)

接著連線上去上傳即可

![](images/部署K8S節點集群Docker/2025-10-28-23-40-02-image.png)

確認每台機器都有安裝包

![](images/部署K8S節點集群Docker/2025-10-28-23-40-42-image.png)

接著先解壓縮 `tar -xf docker.tar.gz`如下圖

![](images/部署K8S節點集群Docker/2025-10-28-23-41-57-image.png)

接著我們進去目錄中輸入指令安裝 `yum install -y ./*.rpm`

![](images/部署K8S節點集群Docker/2025-10-28-23-45-28-image.png)


