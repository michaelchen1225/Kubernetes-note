# Kustomize

## Kustomize 簡介

一個專案可能由多份 yaml 組成，例如一個網站可能使用

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

設定好 kustomization.yml 後，使用以下指令即可部署訂製後的 yaml：

```bash
# kubectl 原生支援 kustomize，不須額外安裝
kubectl apply -k /path/to/kustomization.yml
```

or

```bash
# 需要先安裝 kustomize 工具
kustomize build /path/to/kustomization.yml | kubectl apply -f -
```

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

> 專案名稱為「my-web」，其中會用到 mongodb 和我們自己開發的 web：

```plaintext
my-web/
│
├── mongo/
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
  * ../mongo/base/kustomization.yaml  <-- statefulset.yaml + 通用設定
  * ../myapp/base/kustomization.yaml  <-- deployment.yaml & service.yaml + 通用設定
  * deployment-patch.yaml             <-- dev 補丁
  * statefulset-patch.yaml            <-- dev 補丁

2. 加入針對 dev 環境的設定              <-- dev 設定        

若執行 `kubectl apply -k overlays/dev/kustomization.yaml`，最終會被部署的 yaml 為：

* statefulset.yaml：通用設定 + dev 補丁 + dev 設定
* deployment.yaml：通用設定 + dev 補丁 + dev 設定
* service.yaml：通用設定 + dev 設定

> kustomization.yaml 是可以互相引用的，若設定有重疊，來源端的設定會被覆蓋。









![alt text](image.png)


## Try it out

The project structure is as follows:

```plaintext
kustomize-demo
├── base
│   ├── backend
│   │   ├── base-back.yml
│   │   ├── kustomization.yml --> resources: base-back.yml
│   └── frontend
│       ├── base-front.yml
│       └── kustomization.yml --> resources: base-front.yml
└── overlays
    ├── dev
    │   └── kustomization.yml
    └── prod
        └── kustomization.yml
```

Our goal：

```plaintext
prod-env：
    namespace: prod
    frontend-deployment：4 replicas, image: nginx:latest, name: prod-front
    backend-deployment：4 replicas, image: redis, name: prod-back

dev-env：
    namespace: dev
    frontend-deployment：1 replicas, image: nginx:1.26.3, name: dev-front
    backend-deployment：1 replicas, image: redis, name: dev-back
```

* create a new directory

```bash
mkdir -p kustomize-demo
cd kustomize-demo
```

* create the basic kustomize structure
```bash
mkdir -p base/frontend && mkdir -p base/backend
mkdir -p overlays/prod && mkdir -p overlays/dev
```
> base: contains the base resources, here we have two different applications and two different environments

> overlays: contains the overlays yaml files



* create a deployment and a service yaml as base resources

```bash
kubectl create deployment frontend --image=nginx --dry-run=client -o yaml > base/frontend/base-front.yml
kubectl create deployment backend --image=redis --dry-run=client -o yaml > base/backend/base-back.yml
```

* create a kustomization file for each application

```bash
vim base/frontend/kustomization.yml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: # here you can add template yaml or other kustomization files
- base-front.yml # template yaml must be in the same directory with kustomization file
# if you have other kustomization files, you can add them here(relative path)
```

```bash
cp base/frontend/kustomization.yml base/backend/kustomization.yml
sed -i 's/base-front.yml/base-back.yml/g' base/backend/kustomization.yml
```

* create the overlays for prod environment：

```bash
vim overlays/prod/kustomization.yml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod

resources: # include other kustomization files(relative path)
- ../../base/frontend
- ../../base/backend

replicas: 
- name: frontend
  count: 4
- name: backend
  count: 4

images:
- name: nginx
  newName: nginx # if you want to use other image, you can change it here
  newTag: latest # change the image tag here
```

* create the overlays for dev environment：

```bash
vim overlays/dev/kustomization.yml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
- ../../base/frontend
- ../../base/backend

replicas:
- name: frontend
  count: 1
- name: backend
  count: 1

images:
- name: nginx
  newName: nginx
  newTag: 1.26.3
```

* test the kustomize for both environments：
```bash
kustomize build overlays/prod
kustomize build overlays/dev
```

* if everything is correct, you can apply the resources to the cluster
```bash
kubectl create ns prod
kubectl create ns dev
kustomize build overlays/prod | kubectl apply -f -
kustomize build overlays/dev | kubectl apply -f -
```

https://weii.dev/kustomize/

## to do

https://ithelp.ithome.com.tw/articles/10355999

helm with kustomize & argocd