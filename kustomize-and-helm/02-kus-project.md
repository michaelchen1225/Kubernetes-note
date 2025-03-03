## Kustomize 的專案結構

一個使用 kustomize 管理的專案目錄，建議使用以下結構：

* **base 目錄**：

  * resources：放置「樣版 yaml」，是每個環境都會用到的資源。

  * kustomization.yml：引入需要的`樣板 yaml` + 加入`通用設定`。

* **overlays 目錄**：
  
  * 環境子目錄：不同的部署環境以「子目錄」區分，例如 overlays/dev、overlays/prod。

    * other resources：放置「某些環境才會用到」的 resources yaml，例如只有 prod 環境需要額外的 Ingress 設定，則就會有一個 overlays/prod/ingress.yml。

    * patches：放置各種的「補丁 yaml」，針對 base 目錄下的樣本 yaml 做修改。

    * kustomization.yml：引入 `base 目錄下的 kustomization.yml` + 套用 `patches` + 加入`針對性設定` + 引入 `other resources` 
  
  > patches v.s 針對性設定：pathces 是額外的 yaml 檔被引入到 kustomization.yml，而「針對性設定」是**直接寫**在 kustomization.yml 中。由於某些設定並不支援在 kustomization.yml 中直接寫，因此需要透過 patches 來修改。

  > 舉例而言，memory limit & request 就需要用 patches 來修改，而 image 則可以直接在 kustomization.yml 中指定。


我們來看一個專案的結構範例：

> 專案名稱為「my-web」，其中會用到 redis 和我們自己開發的 web：

```plaintext
my-web/
│
├── redis/
│   ├── base/
│       ├── statefulset.yaml  <-- 樣板yaml
│       └── kustomization.yaml <-- statefulset.yaml + 通用設定
│
├── myapp/
|   ├── base/
|       ├── deployment.yaml  <-- 樣板yaml
|       ├── kustomization.yaml  <-- deployment.yaml & service.yaml + 通用設定
|       └── service.yaml  <-- 樣板yaml
|  
├── overlays/
    ├── dev/
    │   ├── deployment-patch.yaml <-- 補丁yaml
    │   ├── statefulset-patch.yaml <-- 補丁yaml
    │   └── kustomization.yaml 
    │
    └── prod/
        ├── deployment-patch.yaml
        ├── statefulset-patch.yaml
        └── kustomization.yaml
```

當我們想要針對 dev 環境作部署時，overlays/dev/kustomization.yaml 應該要：

1. 引入：
  * ../redis/base/kustomization.yaml  <-- statefulset.yaml + 通用設定
  * ../myapp/base/kustomization.yaml  <-- deployment.yaml & service.yaml + 通用設定
  * deployment-patch.yaml             <-- dev 補丁
  * statefulset-patch.yaml            <-- dev 補丁

2. 加入針對 dev 環境的設定              <-- dev 設定        

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../redis/base
- ../../myapp/base

patches:



若執行 `kubectl apply -k overlays/dev/kustomization.yaml`，最終會被部署的 yaml 為：

* statefulset.yaml：通用設定 + dev 補丁 + dev 設定
* deployment.yaml：通用設定 + dev 補丁 + dev 設定
* service.yaml：通用設定 + dev 設定

> kustomization.yaml 是可以互相引用的，若設定有重疊，來源端的設定會被覆蓋。(通常是 overlay 去引用 base 的 kustomization，所以有重疊的部分以 overlay 為主)