# Kustomization 語法彙整

## 目錄

* [名詞解釋](#名詞解釋)

* [環境準備](#環境準備)

* [Transformer：通用設定](#Transformer)
  * [commonLabels](#commonlabels統一新增-label)
  * [commonAnnotations](#commonannotations統一新增-annotation)
  * [namespace](#namespace統一指定-namespace)
  * [namePrefix](#nameprefix統一新增-prefix)
  * [nameSuffix](#nameSuffix統一新增-Suffix)
  * [image](#image統一修改-Image)
  * [Custom Transformer --- Configuration](#custom-transformer-----configuration)

* [Patch：以 Strategic Merge Patch 為例](#patch以-strategic-merge-patch-為例)
  * [Patch 的基本觀念](#patch-的基本觀念)
  * [merge --- 移除整個 Map](#merge-----移除整個-map)
  * [merge --- 移除特定 key value](#merge-----移除特定-key-value)
  * [merge --- 修改 value](#merge-----修改-value)
  * [merge --- 新增](#merge-----新增)
  * [replace --- 重新改寫](#replace-----重新改寫)
  * [一次 Patch 多個 Resource](#一次-patch-多個-resource)

* [configMapGenerator、secretGenerator](#configmapgeneratorsecretgenerator)
  * [configMapGenerator --- 生成內涵檔案的 ConfigMap](#configmapgenerator-----生成內涵檔案的-configmap)
  * [configMapGenerator --- 生成帶有 key-value 的 ConfigMap](#configmapgenerator-----生成帶有-key-value-的-configmap)
  * [從 .env 檔案生成 ConfigMap](#從-env-檔案生成-configmap)
  * [secretGenerator --- 生成 Secret](#secretgenerator-----生成-secret)

## 名詞解釋

* Transformer：為**全部**的樣本 yaml 套用某種通用變更。kustomize 有內建許多的 Transformer 可以使用，例如 commonLabels、commonAnnotations、namespace、namePrefix、nameSuffix、image 等等。

* patches：使用 **JSON6902 Patch** 或 **Strategic Merge Patch** 來修改**特定**的樣本 yaml。設定方式可分兩種：
  1. Inline：直接寫在 kustomization.yml 中。
  2. patches：寫法同 Inline，只不過放在另外的 yaml 檔案中，再透過 `patches.path` 引入至 kustomization.yml。

* 樣本 yaml 以下稱為 **Resource**。

> Key：通用設定用 common Transformer，個別設定用 patches。

## 環境準備

下載測試所需的 Resource yaml (不含 kustomization.yml)：

```bash
git clone https://github.com/michaelchen1225/kustomize-simple-demo.git
```

> 載下來之後可以先看看 yaml 裡都設定了什麼。

## Transformer

### commonLabels：統一新增 Label

`commonLabels` 會在「所有有關 Label」的地方打上指定的 Label：

* 如果 Label 欄位不存在，則會新增該欄位
* 如果 Label 欄位已存在，則會附加新的 Label。

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- deploy.yaml
- svc.yaml

commonLabels:
  env: new-prod
  project: new-project
```

```bash
kubectl kustomize .
```

觀察輸出結果可發現：

* 所有 Resouce 的 metadata.Labels 都多了 env: new-prod 和 project: new-project。

* pod.yaml 本來沒有 metadata.Labels，所以 `commonLabels` 直接新增了該欄位。

* 除了 metadata.Labels 之外，所有關於 label 的地方都會添加 `commonLabels` 的設定，例如 Service、Deployment 的 Selector、Deployment 的 tmeplate.metadata.Labels 等等。

### commonAnnotations：統一新增 Annotation

`commonAnnotations` 的用法與 `commonLabels` 類似，只是作用對象是 Annotation。

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- deploy.yaml
- svc.yaml

commonAnnotations:
  imageregistry: "https://hub.docker.com/"
```

```bash
kubectl kustomize .
```

### namespace：統一指定 Namespace

`namespace` 會在所有 Resource 的 metadata.namespace 欄位指定 Namespace。

* 若 Resource 本來就有 metadata.namespace，則會被覆蓋。
* 若 Resource 本來沒有 metadata.namespace，則會新增該欄位。

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- deploy.yaml
- svc.yaml

namespace: new-prod
```

### namePrefix：統一新增 Prefix

`namePrefix` 會在所有 Resource 的 metadata.name 欄位加上 Prefix。

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- deploy.yaml
- svc.yaml

namePrefix: prod-
```
```bash
kubectl kustomize .
```

### nameSuffix：統一新增 Suffix

原理同 `namePrefix`，只是加在後面。

```yaml
# kustomization.yml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- deploy.yaml
- svc.yaml

nameSuffix: -prod
```

```bash
kubectl kustomize .
```

### image：統一修改 Image

格式：

```yaml
image:
- name: <old image name>
  newName: <new image name> # 如果只想改 tag，則不用寫這行
  newTag: <new image tag> # 若 tag 為數值，需加上引號 ""
```

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- pod-2.yaml
- deploy.yaml
- deploy-2.yaml

images:
- name: nginx  
  newTag: 1.19.10
- name: my-web 
  newName: my-new-web
  newTag: v1
```

```bash
kubectl kustomize .
```

觀察結果可以發現：

* 所有 Resource 中的 image 只要用 nginx，會被加上新的 tag 1.19.10。
* 所有 Resource 中的 image 只要用 my-web，會被改成 my-new-web:v1。

### Custom Transformer --- Configuration

如果有自己的 CRD 也想要套用 Transformer，得先透過 configuration 告訴 Transformer 要套用在哪裡。

範例：(https://github.com/kubernetes-sigs/kustomize/blob/master/examples/transformerconfigs/images/README.md)


## Patch：以 Strategic Merge Patch 為例

### Patch 的基本觀念

* Patch 有兩種寫法：JSON6902 Patch 和 Strategic Merge Patch。

* Patch 能在 Inline 或是透過 patches.path 引入。

* 為了維持 kustomization.yml 的簡潔，應該用 patches.path 來引入 Patch yaml

* 指定 patch 的目標：target
  
  ```yaml
  patches:
  - path: <relative path to file containing patch>
    target:
      group: <optional group>
      version: <optional version>
      kind: <optional kind>
      name: <optional name or regex pattern>
      namespace: <optional namespace>
      labelSelector: <optional label selector>
      annotationSelector: <optional annotation selector>
  ```

  * 使用 Strategic Merge Patch 時，不指定 target 會以 patch yaml 的資訊為準。
  * 使用 JSON6902 Patch 時，必須指定 target。

* 常見的 Strategic Merge Patch 操作：

  * **merge**：套用 patch 時，沒有指定的欄位會保留原狀、新欄位會被新增、key 值相同的 value 值會被取代。(底下看完範例後會比較清楚)


  * **replace**：常用在需要「整個重新改寫」的情況。與 merge 唯一不同的是，patch 中沒設定的地方會被直接移除。

  > 之所以不介紹 delete，是因為 merge 能使用 null 值達到相同效果。

以下會逐一示範上述操作，使用 Strategic Merge Patch，並且透過 patches.path 引入。

### merge --- 移除整個 Map  

**範例**：把 deploy.yaml 的 metadata.labels 移除。

* 在寫 patch 時有個訣竅：只要留下 target 資訊(apiVersion、kind、metadata.name)和要修改的欄位即可，其他通通不用寫(沒有指定的欄位會保留原狀)。

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels: null
  name: deploy
```

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deploy.yaml

patches:
- path: deploy-patch.yaml
```

> 測試：kubectl kustomize .


### merge --- 移除特定 key value

**範例**：把 deploy.yaml 的 `app: deploy` 移除。

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: null
  name: deploy
```

> kustomization.yml 同上

### merge --- 修改 value

* 修改 deploy.yaml 的：
  * label：env：dev --> env: prod
  * container image：my-web --> my-new-web

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    env: prod
  name: deploy
spec:
  template:
    spec:
      containers:
      - image: my-new-web
        name: my-web
```
> kustomization.yml 同上

### merge --- 新增

**範例**：給 deploy.yaml 添加新 label，並新增一個 container

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    new: label  # 新 label
  name: deploy
spec:
  template:
    spec:
      containers: # 新 container
      - image: new-container
        name: new-container
```

> kustomization.yml 同上

### replace --- 重新改寫

replace 適合用在要「整段 Map/list 重寫」的情況。replace 與 merge 的差別在於，patch 中沒提到的地方會被直接移除，而非保留原狀。

**範例**：重寫 deploy.yaml 的 label 和 template，只留下 `new: label` 與 `new-container`。

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  $patch: replace
  labels:
    new: label  # 新 label
  name: deploy
spec:
  template:
    spec:
      $patch: replace
      containers: # 新 container
      - image: new-container
        name: new-container
```   

> kustomization.yml 同上

這是一個混合 merge 和 replace 的例子，可以看到：

* `$patch: replace` 寫在 labes 和 containers 之上，表示整個 labels 和 containers 都會被重新改寫。
* 沒有標示 `$patch: replace` 的地方使用預設的 merge，因此保留原狀。


### 一次 Patch 多個 Resource

> Offical Doc：(https://github.com/kubernetes-sigs/kustomize/blob/master/examples/patchMultipleObjects.md)

透過在 target 的設定，我們能讓一個 patch.yaml 套用在多個 Resource 上。target 的設定格式如下：

```yaml
  patches:
  - path: <relative path to file containing patch>
    target:
      group: <optional group>
      version: <optional version>
      kind: <optional kind>
      name: <optional name or regex pattern> # 支援 regex，例如 name: foo*
      namespace: <optional namespace>
      labelSelector: <optional label selector>
      annotationSelector: <optional annotation selector>
```

**將 deploy.yaml** 與 **deploy-2.yaml** 都打上 `new: label`，並且加入一個 sidecar container。

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    new: label 
  name: the-name-is-not-important
spec:
  template:
    spec:
      containers:
      - image: sidecar
        name: sidecar
```
> 由於 patch 的目標會在 kustomization.yml 中用 `target` 指定，所以 patch.yaml 中的 name 可以隨便寫。

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deploy.yaml
- deploy-2.yaml

patches:
- path: deploy-patch.yaml
  target:
    kind: Deployment
```

## configMapGenerator、secretGenerator

configMapGenerator 可以用來來生成 ConfigMap、secretGenerator 可以用來生成 Secret。

### configMapGenerator --- 生成內涵檔案的 ConfigMap

* 建立測試目錄：

```bash
$DEMO_HOME=$(mktemp -d)
cd $DEMO_HOME
```

* 建立檔案：

```bash
cat <<EOF >config-file.properties
this is a config file
EOF
```

* 撰寫 pod.yaml，引用需要被生成的 ConfigMap：

```bash
cat <<EOF >pod.yaml
#pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: my-volume
      mountPath: /etc/config
  volumes:
  - name: my-volume
    configMap:
      name: my-configmap
EOF
```

* 撰寫 kustomization.yml，生成 ConfigMap：

```yaml
#kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

configMapGenerator:
- name: my-configmap
  files:
  - comfig-file.properties
```

* 實際把 pod、configMap 跑起來看看：

```bash
kubectl apply -k .
```

* 可以在 my-pod 中的 /etc/config 看到 config-file.properties 的內容：

```bash
kubectl exec -i my-pod -- cat /etc/config/config-file.properties
```

* 清除資源：

```bash
kubectl delete -k .
```

### configMapGenerator --- 生成帶有 key-value 的 ConfigMap

* kustomization.yml：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

configMapGenerator:
- name: my-configmap
  literals:
  - NAME=Michael
  - Age=25

generatorOptions:
 disableNameSuffixHash: true  # 關閉 comfigMap 生成後加上的 hash
 labels:  # 給 ConfigMap 加上 Label
  demo: configMapGenerator
```

* 修改 pod.yaml，讓 pod 從 ConfigMap 中取得環境變數：

```bash
cat <<EOF >pod.yaml
apiVersion: v1
kind: Pod

metadata:
  name: my-pod

spec:
  containers:
  - name: my-container
    image: nginx
    envFrom:
    - configMapRef:
        name: my-configmap
EOF
```

* 實際把 pod、configMap 跑起來看看：

```bash
kubectl apply -k .
```

* 可以在 my-pod 中的環境變數看到 NAME=Michael 和 Age=25：

```bash
kubectl exec -i my-pod -- env
```

### 從 .env 檔案生成 ConfigMap 

* 若環境變數太多，可以先寫在 .env 檔案中，再透過 configMapGenerator 生成 ConfigMap：

```bash
cat <<EOF >.env
NAME=Michael
Age=25
Birthday=1996-12-25
EOF
```

* kustomization.yml：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

configMapGenerator:
- name: my-configmap
  envs:
  - .env
```

### secretGenerator --- 生成 Secret

用法和 configMapGenerator 基本相同，把 configMapGenerator 改成 secretGenerator 即可：

```bash
echo -n "Michael" > user.txt
echo -n "123456" > password.txt
```

```yaml
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

secretGenerator:
- name: my-secret
  files:
  - user.txt
  - password.txt

generatorOptions:
  disableNameSuffixHash: true
  labels:
    demo: secretGenerator
```

```bash
kubectl kustomize .
```

## 小結

今天介紹的基本語法，足以應付一般的需求。不過今天的環境其實很單純，只有一個目錄。下一篇會介紹一個專案是如何用 kustomize 管理的。 