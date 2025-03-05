# Kustomization 語法彙整

## 名詞解釋

* Transformer：為**全部**的樣本 yaml 套用某種變更。kustomize 有內建許多的 Transformer 可以使用，例如 commonLabels、commonAnnotations、namespace、namePrefix、nameSuffix、image 等等。

  * configuration：額外寫一個 yaml，指定 Transformer 更改的路徑。(底下會有範例)

* patches：使用 **JSON6902 Patch** 或 **Strategic Merge Patch** 來修改**特定**的樣本 yaml。設定方式可分兩種：
  1. Inline：直接寫在 kustomization.yml 中。
  2. patches：寫法同 Inline，只不過放在另外的 yaml 檔案中，再透過 `patches.path` 引入至 kustomization.yml。

* 樣本 yaml 以下稱為 **Resource**。

> Key：通用設定用 common Transformer，個別設定用 patches。

## 環境準備

測試時會使用的樣本 yaml 已經寫好了，clone 下來之後僅需修改 kustomization.yml 即可。

```bash
git clone 
```

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

**目的**：把 deploy.yaml 的 metadata.labels 移除。

* 在寫 patch 時有個訣竅：只要留下 target 資訊(apiVersion、kind、metadata.name)和要修改的欄位即可，其他通通不用寫(因為沒有指定的欄位會保留原狀)。

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

### merge --- 移除特定 key value

* 把 deploy.yaml 的 app: deploy 移除。

```yaml
# deploy-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: null
  name: deploy
```

kustomization.yml 同上

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

### replace

### 混合使用

### 一次 Patch 多個 Resource