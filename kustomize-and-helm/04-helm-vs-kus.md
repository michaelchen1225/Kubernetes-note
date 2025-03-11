# Helm vs Kustomize

## 目錄

概念複習：

* [Helm](#helm)

* [Kustomize](#kustomize)

Helm vs Kustomize：

* [Helm vs Kustomize](#helm-vs-kustomize-1)

Helm + Kustomize：

* [Why using Helm with Kustomize](#why-using-helm-with-kustomize)

* [Helm + Kustomize 實作](#helm--kustomize-實作)
  * [準備 Helm Chart](#準備-helm-chart)
  * [方法 1：helm template + kustomize](#方法-1helm-template--kustomize)
  * [方法 2：helm install --post-renderer + kustomize](#方法-2helm-install---post-renderer--kustomize)
  * [補充：helm template --post-renderer + kustomize](#補充helm-template---post-renderer--kustomize)

---

前幾篇文章分別介紹了 helm 與 kustomize，底下簡單複習一下：

### Helm

Helm 是一個 package manager，類似於 apt 或 yum，但是專門用於 Kubernetes。

在 Helm  中，一個 package 被稱為「Chart」。在打包 chart 之前，需要先有一個基本的 helm chart 目錄，標準結構如下：

```bash
# 生成標準 helm chart 目錄
helm create helm-demo
```

```plaintext
helm-demo
|-- Chart.yaml
|-- charts
|-- templates
|   |-- resource.yaml
|   |-- NOTES.txt
|-- values.yaml
```

* Chart.yaml：用來描述該 chart 的基本資訊，例如 chart 的名稱、版本、描述等。
* charts：放置「子 chart」的目錄，通常有相依性的 chart 會放在這裡，這個目錄預設是空的。
* templates: 放置 k8s 資源 yaml 的目錄，
* NOTES.txt：這個檔案會在部署 chart 時顯示，提醒使用者一些重要資訊。
* **values.yaml**：其中定義的 key-value 會以「go template」的方式套用到 templates 目錄裡的 yaml 中，達到客製化的效果。

把通用的 template yaml 和 values.yaml 定義好後，就可以打包成一個 **Chart**：

```bash
helm package ./helm-demo
```

打包後會產生一個 `helm-demo-x.x.x.tgz` 的 chart，也因為打包的關係，helm chart 可以做到版本的更迭，並透過 `helm install` 來部署、`helm upgrade` 來更新、`helm uninstall` 來刪除。

> 有用過 apt 或 yum 的人應該會覺得 helm 的操作方式很熟悉。

### Kustomize

Kustomize 是一個 confiuration management tool，在 kubectl 中原生支援，用於客製化 k8s 資源 yaml。

Kustomize 只需要透過 yaml 就可以描述客製化的規則，而非使用 template language，較容易閱讀。

Kustomize 使用一個名為 `kustomization.yaml` 的檔案來描述客製化的規則，而一個專案目錄下可以有許多 kustomization.yaml，彼此互相引用、組合，最終形成 k8s 資源 yaml。

一個理想的 kustomize 專案目錄結構如下：

```plaintext
kustomize-demo
|-- base
|-- components
|-- overlays
```

* base：放置全環境通用的 yaml 及設定
* components：將設定模組化，供不同的環境引用
* overlays：針對不同的環境進行設定

生成需要的 yaml 非常簡單，只須找到需要的 kustomization.yaml，然後透過 `kubectl apply -k <parent-dir/of/kustomization.yaml>` 即可。

### Helm vs Kustomize

兩者的差異性可以體現於：


* **角色不同**：Helm 是 package manager，Kustomize 是 configuration management tool。單純使用 Kustomize，無法做到如 Helm 版本更迭、依賴管理等功能，在專案的分享上也沒有 Helm 方便。

* **語法不同**：Helm 使用 go template，Kustomize 使用 yaml。對於不熟悉 K8s 的人來說，一份註解清晰的 valus.yaml 會使的 Helm 更容易閱讀 & 客製化；而對於熟悉 K8s 的人來說，Kustomize 能直接以 yaml 來描述客製化規則，更加容易閱讀。

* **彈性不同**：若 Chart 裡面沒有你需要的資源(ex. configMap)，除了找到 Chart 的原作者、請他增加，不然也無計可施；但 Kustomize 可以透過 overlay 的方式，將自己要的資源定義好，再引入到 kustomization.yaml 中。另外在專案架構的彈性上，kustomize 只需掌握好 base、components、overlays 的關係，就能任意分出子目路來分類不同設定。

* **複雜性不同**：如果要增加修改設定值，Helm 必須在茫茫 template 中找到對應的 yaml 進行修改，再同步更新 values.yaml；而 Kustomize 因為架構簡單易讀，可以很容易就找到修改目標，並使用單純的 yaml 語法進行修改。

* **易用性不同**：Helm 有一套完整的操作流程，並且有許多現成的 chart repo 可以使用，但是對於不熟悉 Helm 的人來說，可能會覺得操作流程較為複雜；而 Kustomize 則是原生支援的 kubectl 子指令，對於熟悉 kubectl 的人來說，操作流程較為簡單。

總而言之，如果你的專案相當龐大，有分享、版控的需求，Helm 加上註解清晰的 values.yaml 就是不二之選，因為使用者面對如此複雜的設定，只需閱讀 values.yaml 即可；但如果專案只是內部使用，且對於 yaml 的閱讀比較熟悉，就可以選擇相對易讀、易用、彈性高的 Kustomize。

> 表格整理

| 比較項目 | Helm | Kustomize |
|----------|------|-----------|
| **角色** | Package manager，提供版本管理、依賴管理，適合分享與覆用 | Configuration management tool，無法版本管理，較不方便分享 |
| **語法** | 使用 Go template，需學習模板語法 | 使用 YAML，直接描述客製化規則 |
| **彈性** | 若 Chart 缺少資源，需請求原作者修改或自行維護 | 可透過 overlay 自行加入資源，架構靈活 |
| **複雜性** | 需修改 template 並同步更新 values.yaml，較為繁瑣 | 架構簡單易讀，直接修改 YAML 即可 |
| **易用性** | 完整操作流程，有豐富的 Chart repo，但入門較複雜 | 原生支援 kubectl，對熟悉 kubectl 的人較簡單 |
| **適用情境** | 適合大規模專案，需要分享與版本管理 | 適合內部專案，熟悉 YAML 的使用者更易上手 |

### Why using Helm with Kustomize

其實 Helm 與 Kustomize 是可以搭配使用的，搭配方式如下：

> 使用 Helm 產生 YAML，再使用 Kustomize 進行修改。

會需要這樣搭配的原因有：

* 使用者無法控制 Chart：雖然 Helm 可以讓使用者方便的拉取專案，但使用者如果想自行修改、增加內容是無法透過 Helm 完成的。

* 需要建立 Secret、ConfigMap：這些敏感資料不應該直接放在 Helm Chart 中，可以透過 Kustomize 在後續補上。

* 為了使 Helm chart 的通用性，一般不會將統一的 namespace、label 直接寫在 Chart 中，而這可以使用 Kustomize 在部署時設定。

* 若要針對不同部署環境進行微調，同樣可以先用 Helm 生成基本的 YAML，再透過 Kustomize 進行修改。

> 總而言之，當無法直接修改 Chart 或是需要增加設定時，Helm + Kustomize 是一個不錯的選擇。

在實際操作上，Helm + Kustomize 有兩種搭配方式：

1. 使用 `helm template` 生成 YAML 清單，再 Kustomize 進行修改。這種方法無法使用 Helm 的 Release 管理功能。
2. 使用 `helm install`（或 `helm upgrade --install`），搭配 `--post-renderer` flag 來指定 Kustomize 進行修改。這種方式可以保留 Helm 的 Release 管理功能，同時也能利用 Kustomize 來調整配置。


### Helm + Kustomize 實作 

#### 準備 Helm Chart

* 初始化一個 Helm Chart：

```bash
DEMO_HOME=$(mktemp -d)
helm create $DEMO_HOME/helm-demo
cd $DEMO_HOME/helm-demo
```

* 先清空 templates 目錄：

```bash
rm -rf templates/*
```

* 撰寫 Chart.yaml：

```bash
cat <<EOF > Chart.yaml
apiVersion: v2
name: helm-demo
description: helm-with-kustomize
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
```

* 創建一個 Pod 作為 template：

```bash
cat <<EOF > templates/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: "{{ .Values.image.name }}"
EOF
```

* 撰寫 values.yaml：

```bash
cat <<EOF > values.yaml
image:
  name: nginx
EOF
```

* 驗證 Helm Chart 是否正確：

```bash
helm template . --values values.yaml
```

#### 方法 1：helm template + kustomize

OK，現在我們已經有一個簡單的 Helm Chart 了。不過這個 Chart 無法讓使用者客製化 image tag，以下用 Kustomize 來解決這個問題：

* 新增一個給 kustomize 的目錄：

```bash
mkdir $DEMO_HOME/kustomize-demo
cd $DEMO_HOME/kustomize-demo
```

* 撰寫 kustomization.yaml：

```bash
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - resources.yaml 

images:
  - name: nginx
    newTag: 1.19.2
EOF
```

* 先測試看看 `helm template + kustomize` 的效果：

```bash
# 在 kustomize-demo 目錄
helm template $DEMO_HOME/helm-demo --values $DEMO_HOME/helm-demo/values.yaml > resources.yaml
kubectl kustomize .
```

> image tag 已被修改成 1.19.2。

#### 方法 2：helm install --post-renderer + kustomize

另外一種方式是在 Helm 中使用 `--post-renderer`，這個 option 會附帶一個可執行檔，在 Helm 正式發布 release 前會先執行這個檔案，對 YAML 進行預渲染：

* 撰寫可執行檔 `kustomize.sh`：

```bash
cat <<EOF > kustomize.sh
#!/bin/bash
cat > resources.yaml
kubectl kustomize .
EOF
```

* 給 `kustomize.sh` 執行權限：

```bash
chmod u+x kustomize.sh
```

* 使用 `helm install` 並指定 `--post-renderer`：

```bash
helm install helm-demo $DEMO_HOME/helm-demo --values $DEMO_HOME/helm-demo/values.yaml --post-renderer $DEMO_HOME/kustomize-demo/kustomize.sh
```

> 在這個指令中，helm install 會先將原本的 YAML 丟給 kustomize.sh 處理，而 kustomize.sh 裡的 `cat > resources.yaml` 負責產生 resources.yaml，再透過 `kubectl kustomize .` 進行修改，最終執行 helm install。

* 現在就可以用 `helm list` 查看 release 了：

```bash
helm list
```
```plaintext
root@ubuntu:/tmp/tmp.NuKfekzVqi/kustomize-demo# helm list
NAME                           	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART                                 	APP VERSION
helm-demo                      	default  	1       	2025-03-11 10:59:48.691947904 +0800 CST	deployed	helm-demo-0.1.0                       	1.0.0   
```

* 確認 Pod 的 image tag 是否正確：

```bash
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'
```
```plaintext
nginx:1.19.2
```

#### 補充：helm template --post-renderer + kustomize

helm template 也可以使用 `--post-renderer` 來產生 YAML：

```bash
helm template $DEMO_HOME/helm-demo --values $DEMO_HOME/helm-demo/values.yaml --post-renderer $DEMO_HOME/kustomize-demo/kustomize.sh
```

### 總結

在專案越來越龐大的時候，Helm 可以用一份 values.yaml 讓使用者快速客製化，如果客製化內容並沒有定義在 values.yaml 中，那麼就可以透過 Kustomize 來進行修改。

### Reference

* [When and How to Use Helm and Kustomize Together](https://trstringer.com/helm-kustomize/)