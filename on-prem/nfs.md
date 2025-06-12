# NFS StorageClass 

> 以下實作如果在 killercoda 上執行，可能會遇到一些權限問題，建議在自己的虛擬機上操錯。

這裡選用 [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) 作為 storageClass 的 provisioner，所以我們得先建置一個 nfs server：

* 在 Master Node 安裝 nfs server 與 nfs client：
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

* 在**所有**的 Node 安裝 nfs client：
```bash
sudo apt install -y nfs-common
```

---
>**注意**

如果你檢查剛剛安裝的 nfs-common 是 inactive (dead)，如下：
  
```bash
systemctl status nfs-common
```
```text
● nfs-common.service
     Loaded: masked (Reason: Unit nfs-common.service is masked.)
     Active: inactive (dead)
```

先 unmask nfs-common：
```bash
sudo systemctl unmask nfs-common
```

如果 nfs-common 還是顯示 masked， 這有可能因為 nfs-common 的服務檔被連結到 /dev/null 了，如下：
```bash
file /lib/systemd/system/nfs-common.service
```
輸出：
```text
/lib/systemd/system/nfs-common.service: symbolic link to /dev/null
```

所以刪除連結檔，重啟服務即可：
```bash
sudo rm /lib/systemd/system/nfs-common.service
sudo systemctl daemon-reload
sudo systemctl restart nfs-common
```
最終要確定 nfs-common 是 active (running) 狀態：
```bash
systemctl status nfs-common
```
```text
● nfs-common.service - LSB: NFS support files common to client and server
     Loaded: loaded (/etc/init.d/nfs-common; generated)
     Active: active (running) since Fri 2024-05-24 15:18:15 UTC; 12s ago
```
***


* 回到 Master Node 上，建置一個要分享的目錄：
```bash
sudo mkdir -p /data/k8s
sudo chown nobody:nogroup /data/k8s
sudo chmod 777 /data/k8s
```

* 查看主機所在網域：
```bash
hostname -I
```
輸出：
```text
192.168.132.1
```

* 將要分享的目錄加入到「/etc/exports」：
```bash
echo -e "/data/k8s\t192.168.132.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
```

* 分享目錄：
```bash
sudo exportfs -av
```
輸出：
```text
exporting 192.168.132.0/24:/data/k8s
```

* 重啟 nfs server：
```bash
sudo systemctl restart nfs-kernel-server
```

* 查看是否有成功分享目錄：
```bash
showmount -e localhost
```
```text
Export list for localhost:
/data/k8s 192.168.132.0/24
```

測試一下分享目錄的效果：

* 建立分享目錄的掛載點：
```bash
sudo mkdir -p /mnt/nfs
```

* 掛載分享目錄：
```bash
sudo mount 192.168.132.1:/data/k8s /mnt/nfs
```

* 建立一個檔案到分享目錄：
```bash
touch /data/k8s/test.txt
```

* 可以在掛載點上看到 test.txt:
```bash
ls /mnt/nfs
```
```text
test.txt
```

完成 nfs server 的建置後，接下來透過 helm 安裝 「nfs-subdir-external-provisioner」:

> 什麼是「helm」？可以參考 [Day 11](https://ithelp.ithome.com.tw/articles/10346850)

* 安裝 helm：
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
``` 

* 加入 nfs-subdir-external-provisioner 的 repo：
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

* 安裝 nfs-subdir-external-provisioner：
```bash
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.132.1 \
    --set nfs.path=/data/k8s \
    --set storageClass.name=nfs-storage
```
```text
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Fri May 24 13:13:22 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

* 查看 storageClass：
```bash
kubectl get sc nfs-storage
```
```text
NAME                	PROVISIONER                                 	RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete      	Immediate       	true               	   17m                 
```

* 查看 provisioner 是否有跑起來：
```bash
kubectl get po | grep nfs
```
```text
NAME                                           	   READY   STATUS 	 RESTARTS   AGE
nfs-subdir-external-provisioner-6444d75b85-56sts   1/1 	   Running   0      	17m
```
> 如果一直卡在 ContainerCreating，但沒有 Error 的 events，可以嘗試重新安裝 nfs-subdir-external-provisioner。如果還是不行，卡在了掛載相關的錯誤，請確認每台 Node 上的 nfs-common 為 active 的狀態。

* 建立一個 PVC：
```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-sc
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
```bash
kubectl apply -f pvc.yaml
```

* 可以看到建立 PVC 後，狀態馬上變成 Bound：
```bash
NAME   	   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound	test-pv                                	   500Mi  	  RWO        	 just-test  	<unset>             	14m
test-sc	   Bound	pvc-b5e66f93-7c2e-4d8b-84af-c507bf2f0829   100Mi  	  RWO        	 nfs-storage	<unset>             	2s
```

* storageClass 建立的 PV 在這裡：
```bash
kubectl get pv
```
```text
NAME   	   STATUS   VOLUME                                 	   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound	test-pv                                	   500Mi  	  RWO        	 just-test  	<unset>             	14m
test-sc	   Bound	pvc-b5e66f93-7c2e-4d8b-84af-c507bf2f0829   100Mi  	  RWO        	 nfs-storage	<unset>             	2s
```

* 刪除 PVC：
```bash
kubectl delete pvc test-sc
```

* 因為 storageClass 的 ReclaimPolicy 是 Delete，所以刪除 PVC 後，PV 也會被刪除：
```bash
kubectl get pv
```
```text
NAME  	  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          	STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
test-pv   500Mi  	 RWO        	Retain       	 Bound	  default/test-pvc  just-test  	   <unset>                      	20m
```

這就是 storageClass 的完整效果：使用者只需要建立 PVC，storageClass 會負責管理 PV 的建立與刪除。