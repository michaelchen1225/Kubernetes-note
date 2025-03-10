# Helm vs Kustomize

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


* **角色不同**：Helm 是 package manager，Kustomize 是 configuration management tool。單純使用 Kustomize，無法做到如 Helm 那樣的版本更迭、依賴管理等功能，在專案的分享上也沒有 Helm 方便。

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

### Helm + Kustomize

Helm 和 Kustomize 可以幫助您更高效地管理 Kubernetes 應用程式。在某些情境下，您可能會想要同時使用 Helm 和 Kustomize，這些情境包括：

當您無法控制 Helm Chart：通常，我們會從 Helm 倉庫拉取並使用他人發布的 Helm Chart。但如果您想修改這些 Chart 內的 YAML 清單（manifests）呢？這時候 Kustomize 可以輕鬆幫助您完成修改。

當需要建立 Secret 和 ConfigMap 資源時：在處理 Secret 和 ConfigMap 時，您通常不希望這些敏感數據直接嵌入 Helm Chart 中。透過 Kustomize，您可以在 Helm 展開（inflate）Chart 之後再建立這些資源。

當需要同時修改多個資源的欄位時：有時候，您可能需要將所有（或部分）資源強制設定到某個命名空間，或是為這些資源統一添加標籤。這些變更通常不應該直接放進 Helm Chart，但 Kustomize 可以在資源上覆蓋這些設定。

兩種主要的使用方式


Helm 和 Kustomize 可以透過兩種主要方式結合使用，但這兩種方式不能同時進行：

* 使用 helm template 生成 YAML 清單，然後使用 Kustomize 進行修改。這種方法的缺點是 Helm 不會管理任何 Release。
* 使用 helm install（或 helm upgrade --install），並在應用到叢集之前使用 Kustomize 修改 YAML 清單。這種方式可以保留 Helm 的 Release 管理功能，同時也能利用 Kustomize 來調整配置。

核心要點
Kustomize 和 Helm 都是強大的 Kubernetes 應用管理工具，雖然它們目標相似，但解決問題的方式不同。
這兩者並不是互相競爭的工具，而是可以搭配使用，以提供更靈活的 Kubernetes 部署方式。
您還可以了解 Spacelift 如何幫助管理 Kubernetes 的複雜性和合規性。Spacelift 的 Kubernetes 支援是基於 Kustomize，並與 kubectl 提供原生整合。