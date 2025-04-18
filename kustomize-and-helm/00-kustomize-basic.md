# Kustomize

## 目錄

* [Kustomize 簡介](#kustomize-簡介)

* [如何部署 kustomize 後的 yaml](#如何部署-kustomize-後的-yaml)

* [kustomization.yml 初探：加上 label & 修改 image](#kustomizationyml-初探加上-label--修改-image)

* [小結](#小結)

### Kustomize 簡介

使用 Kubernetes 部署專案時，該專案可能由多份 yaml 組成，例如一個網站可能使用：

* Deployment 部署

* Service 對外 expose

* ConfigMap 儲存設定檔

* Secret 儲存 image 拉取的 credentials。

* ....

而在現實的使用場景中，開發/修改完一個應用服務後並不會馬上讓使用者使用，可能會先依序部署在 Development 、Testing、Staging 環境，等一切都 OK 後才上 Production 環境。

然而每個環境都可能會有細部的設定需要調整，例如 replicas 數量、image 版本、namespace 等等，我們當然可以手動一個個改，但容易造成人為疏失，而且很沒效率。

如果能將「需要調整的設定」統一寫到一個檔案中，未來更換環境時只需修改這個檔案即可，就能避免上述問題，而這就是 Kustomize 的用途。

**Kustomize** 是一個專門管理 Kubernetes yaml 的工具，其中的核心為 `kustomization.yml`，這個檔案用來設定：

1. 哪些 yaml 會被使用到？(引入樣本 yaml)
2. 這些 yaml 要如何被修改？(針對樣本 yaml 做修改)

之後要部署專案時，我們只需針對不同環境修改 `kustomization.yml` 即可，而不用去改所有 yaml。

### 如何部署 kustomize 後的 yaml

設定好 kustomization.yml 後，使用以下指令即可部署客製後的 yaml：

```bash
# kubectl 原生支援 kustomize，不須額外安裝
# <dir>：kustomization.yml 所在的目錄
kubectl apply -k <dir> 
```

or

```bash
# 需要先安裝 kustomize 工具
kustomize build <dir> | kubectl apply -f -
```

> 安裝 kustomize：

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

### kustomization.yml 初探：加上 label & 修改 image

**事前準備**

建立 Deployment 和 Service 的 yaml 檔：

```bash
kubectl create deployment nginx --image=nginx:latest --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml > service.yaml
```

> 可以先自己看一下這兩份 yaml 的內容。

---

這裡是一份簡單的 kustomization.yml：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: # 引入樣本 yaml
- deployment.yaml
- service.yaml

commonLabels: # 兩個樣本都打上 label
  demo: basic-kustomize

images: # 覆蓋掉所有樣本的 image
- name: nginx # 舊 image
  newName: nginx # 新 image
  newTag: 1.26.3 # 新 tag
```

> 這份 kustomization.yml 引入了 deployment.yaml 和 service.yaml 作為樣本，修改內容是：

* 幫兩者加上 `demo: basic-kustomize` 的 label。
* 把所有叫做 nginx 的 image 都改成 nginx:1.26.3。(如果只是要改 tag，可以不用寫 newName)

---

驗證看看：

```bash
# 僅輸出修改後的 yaml，不會真的部署
kubectl kustomize .
```

```yaml
piVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
    demo: basic-kustomize # 加上的 label
  name: nginx
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
    demo: basic-kustomize 
  type: ClusterIP
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
    demo: basic-kustomize # 加上的 label
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      demo: basic-kustomize
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
        demo: basic-kustomize
    spec:
      containers:
      - image: nginx:1.26.3 # 修改的 image
        name: nginx
        resources: {}
status: {}
```

確認無誤後，就可以部署了：

```bash
kubectl apply -k .
```

刪除剛剛部署的所有資源：

```bash
kubectl delete -k .
```

### Patch

雖然能直接用關鍵字來修改 yaml，但有些情況下我們並非只是單純的「取代」某個值，可能需要「新增」、「刪除」某個欄位，或是一次變更多個欄位。這時就需要用到 **Patch**。

Patch 的方式有兩種：

2. JSON6902 Patch：遵循 [RFC6902](https://tools.ietf.org/html/rfc6902) 的格式，對 yaml 進行修改。

2. Strategic Merge Patch：直接採用 Kubernetes 的 yaml 寫法，說明該如何修改。

我們來看一個簡單的範例：修改 Pod 的 labels。

原始 yaml：

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
```

* Patch：將 lables 改成 `env: prod`：

**JSON6902 Patch**：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

patches: 
- target:
    kind: Pod
    name: nginx
  patch: |-
    - op: replace
      path: /metadata/labels
      value:
        env: prod
```

* op：要進行的操作，這裡是 `replace`，也可以是 `add`、`remove`。

* path：要修改的路徑。

* value：新的值。

**Strategic Merge Patch**：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

patches: 
- patch: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
      labels:
        $patch: replace 
        env: prod
```

> JSON6902 Patch 與 Strategic Merge Patch 的差異在於**寫法**，功能上是一樣的。筆者個人比較偏好 Strategic Merge Patch，因為寫法比較直覺(K8s yaml 的格式)。

上面直接將 patch 寫在 kustomization.yml 中的方式，稱為 **Inline Patch**，但當 patch 較多時會變得雜亂，因此可以將 patch 寫在另外的 yaml 檔案中再引入：

**JSON6902 Patch**：

```yaml
# pod-patch.yaml
- op: replace
  path: /metadata/labels
  value:
    env: prod
```

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

patches: 
- target:
    kind: Pod
    name: nginx
  path: pod-patch.yaml
```



**Strategic Merge Patch**：
```yaml
# pod-patch.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    $patch: replace
    env: prod
```

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

patches: 
- target:
    kind: Pod
    name: nginx
  path: pod-patch.yaml
```

測試之後可以發現與 Inline Patch 的結果是一樣的。


## 小結

本篇介紹了為何需要 Kustomize，以及了解了最基本的 kustomization.yml 設定。[下一篇](01-kus-syntax.md)將會列出許多常用的場景。


