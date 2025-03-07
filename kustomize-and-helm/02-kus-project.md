## Kustomize 的專案結構

## 目錄

* [Kustomization.yml 的基本引用](#kustomizationyml-的基本引用)

* [Overlay 與 Base](#overlay-與-base)

* [Kustomize 常見的專案結構](#kustomize-常見的專案結構)

* [Component](#component)
  * [Component 與 Overlay 的關係：先合併後渲染](#component-與-overlay-的關係先合併後渲染)
  * [Component --- patch 模組化](#component-----patch-模組化)

* [總結](#總結)

## Kustomization.yml 的基本引用

在[上一篇](https://github.com/michaelchen1225/Kubernetes-note/blob/master/kustomize-and-helm/01-kus-syntax.md#%E7%92%B0%E5%A2%83%E6%BA%96%E5%82%99)的語法介紹中，我們只撰寫一份 kustomization.yml，但其實一個專案中可以有多個 kustomization.yml，並相互引用。


kustomization.yml 說白了，代表的就是一個「渲染成果」，最終也是一堆的 yaml 檔。當某份 kustomization.yml 被其他 kustomization.yml 引用時，其實是對「渲染成果」的引用，並進行第二次的渲染。

我們來看一個例子：

* 準備測試環境：

```bash
git clone https://github.com/michaelchen1225/kustomize-with-kustomize.git
cd kustomize-with-kustomize
```

* 我們先在 dev 目錄下建立 kustomization.yml：

```bash
vim dev/kustomization.yml
```

```yaml
# dev/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml
- svc.yaml

commonLabels:
  env: dev
```

```bash
kubectl kustomize dev
```

> pod.yaml 和 svc.yam 都會被加上 `env: dev` 的 Label。

* 接著我們在 prod 目錄下建立 kustomization.yml：

```bash
vim prod/kustomization.yml
```

```yaml
# prod/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../dev # 引用 dev 目錄下的 kustomization.yml
- prod-config.yaml # prod 自己的設定


namespace: prod

commonLabels:
  env: prod

```

```bash
kubectl kustomize prod
```

觀察輸出結果，可以發現：在 prod/kustomization.yml 中，引用了 dev 目錄下的 kustomization.yml，這會對 pod.yaml 和 svc.yaml 進行二次渲染 --> 被指定到 `namespace: prod`，並加上 `env: prod` 的 Label(覆蓋掉首次選染的 `env: dev`)。

如果我們使用 `kubectl apply -k prod`，最終會部署的 yaml 會包括了經過二次渲染的 pod.yaml 和 svc.yaml，以及 prod 自己的 prod-config.yaml。

* 我們將 prod/kustomization.yml 修改一下，讓 `env: dev` 保留下來，但加上新的 label：`version: v1`：

```yaml
# prod/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../dev 
- prod-config.yaml

namespace: prod

commonLabels:
  version: v1
```

```bash
kubectl kustomize prod
```

觀察輸出結果，可以發現與預期相同。總而言之，kustomization.yml 可以互相引用，進行多次的渲染，產生最終的 yaml。

> **注意**：引用其他目錄的 kustomization.yml 時，在 resources 欄位中指定到 parent 目錄即可(不然會報錯)。

## Overlay 與 Base

透過互相引用產生最終 yaml 的方式，其實在 Kustomize 是一種 Overlay 與 Base 的概念：

* **Base**：基礎設定，放置一些通用的 yaml 樣板，例如 Pod、Service、Deployment 等。在 Base 會有一份 kustomization.yml 進行基礎的通用設定。

* **Overlay**：覆蓋設定，通常會引用 Base 的 kustomization.yml，並進行一些針對性的設定，形成最終的 yaml。

> 以上面的例子來說，dev 目錄就是 Base，prod 目錄就是 Overlay。

總而言之，只要一個目錄中有 kustomization.yml，他就能成為其他目錄的 Base。

## Kustomize 常見的專案結構

在一個專案中，可能會有多個應用，每個應用可能會部署到多個環境。這時使用 base + overlays 的結構，可以讓我們針對不同的應用場景進行部署。

為了使專案架構一看就知到哪些是 base、哪些是 overlays，建議直接以 base 和 overlays 命名目錄：


> 專案名稱為「my-web」，其中會用到 redis 和自己開發的 web：

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

這個架構並非強制性的，舉例來說，你也可以將 redis 與 myapp 做成子目錄放置在一個 base 目錄下。一般來說，可以這樣設計專案的目錄結構：

* base 目錄：一定要有 kustomization.yml(才能被 overlay 引用)，並做一些基礎設定。

* overlays 目錄：在底下為不同的使用場景建立子目錄(ex: overlay/dev、overlay/prod)，並在子目錄下建立 kustomization.yml，引用 base 目錄的 kustomization.yml，並針對自身需求進行設定。

> 總之彈性很大，只要清楚的了解專案的目錄結構，就能用各種 kustomization.yml 進行組合，拼湊出最終的 yaml。

舉例而言，現在要針對 dev 環境作部署，可以這樣設定 kustomization.yml：

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../redis/base
- ../../myapp/base

namespace: dev

commonLabels:
  env: dev

patches:
- path: deployment-patch.yaml
  target:
    kind: Deployment
- path: statefulset-patch.yaml
  target:
    kind: StatefulSet
```

如此一來，最終的 yaml 會長這樣：

* 全部的 yaml 被打上 `env: dev` 的 Label。

* 全部的 yaml 部署到 `dev` namespace。

* 所有的 deployment 都會套用 deployment-patch.yaml。

* 所有的 statefulset 都會套用 statefulset-patch.yaml。

## Component

我們來看看底下一種情境：

* 這次的專案有四個部署環境：dev、test、staging、prod。

* 其中除了 dev 之外，其他環境都會部署 `ingress.yaml`

這時，如果我們將 ingress.yaml 放在 base 目錄顯然沒有麼合適：

* 如果我們將 ingress.yaml 放在 base 目錄中，從 dev 環境切換到其他環境時，就要修改 base 目錄的 kustomization.yml 來加入 ingress.yaml、換回 dev 時又要修改來排除 ingress.yaml。這樣非常容易造成人為失誤：假如哪天有人忘記改了怎麼辦？

放在 base 目錄不合適，那放在 overlays 目錄呢？ 就是除了 overlays/dev 之外，其他環境的 overlays 都要加入 ingress.yaml。

但是當 ingress.yaml 更新時，就得手動同步 test、staging、prod 的 ingress.yaml，這也容易造成人為疏失。

就上面介紹的慣例來說，base 就是放置「所有環境都會用到的 yaml」，似乎沒有地方放置「**大多數環**境都會用到的 yaml」，這時就可以使用 Component 的概念：

* components 目錄：放置「大多數環境都會用到的 yaml」，同樣以 kustomization.yml 進行設定。

這樣的好處是，當 Component 更新時，只要更新一次 Component 的 yaml，所有引用到 Component 的 overlays 都會自動更新。當哪天有其他 overlays 也需要引用 Component 時，只要在 kustomization.yml 中加入 Component 的路徑即可。

我們以上面的例子來舉例：

```plaintext
my-web/
│
├── base/
|
├── componets/
|   ├── ingress/
|       ├── ingress.yaml
|       └── kustomization.yaml
| 
├── overlays/
    ├── dev
    ├── test
    ├── staging
    └── prod
```

componet 的 kubeomization.yml：

```yaml
# components/ingress/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1 # 注意這裡是 v1alpha1
kind: Component 

resources:
- ingress.yaml

<其他設定>
```
> 除了 apiVersion 須注意版本、kind 改成 Component 之外，其他設定與 kustomization.yml 一樣。

這時後 test、staging、prod 的 kustomization.yml 就可以這樣設定：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

components:
- ../../components/ingress # 引用 Component

<其他設定>
```

這樣一來，就成功的將 ingress 的相關設定「模組化」，讓不同的 overlays 根據自身需求引用 Component。總之，未來出現「只套用在部分環境」的 yaml 時，在 components 目錄下建立所屬的子目錄並加上 kustomization.yml 即可。


### Component 與 Overlay 的關係：先合併後渲染

* base 與 components 都會被 overlays 引用，但引用時的效果不同：
  * base + overlay：base 中的設定並不會引響到 overlay，base 與 overlay 是「先後渲染」的關係。
  * components + overlay：overlay 會先引用 components 的設定再進行渲染，components 與 overlay 是「先合併後選染」的關係。

很抽象對吧？直接看一個例子：

* 建立三個目錄 --- base、components、overlays：

```bash
mkdir -p /tmp/base
mkdir -p /tmp/components
mkdir -p /tmp/overlays
```

* 在 base 目錄下建立一個 pod.yaml：

```bash
cat <<EOF > /tmp/base/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: base-pod
spec:
  containers:
  - name: my-container
    image: nginx
EOF
```

* 設定 base 的 kustomization.yml：

```bash
cat <<EOF > /tmp/base/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- pod.yaml

commonLabels:
  base: this-is-base
EOF
```

* 在 components 建立一個 pod.yaml：

```bash
cat <<EOF > /tmp/components/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: component-pod
spec:
  containers:
  - name: my-container
    image: nginx
EOF
```

* 設定 components 的 kustomization.yml：

```bash
cat <<EOF > /tmp/components/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- pod.yaml

commonLabels:
  component: this-is-component
EOF
```

* 在 overlays 建立一個 pod.yaml：

```bash
cat <<EOF > /tmp/overlays/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: overlay-pod
spec:
  containers:
  - name: my-container
    image: nginx
EOF
```

* 設定 overlays 的 kustomization.yml，引用 base 與 components：

```bash
cat <<EOF > /tmp/overlays/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base
- pod.yaml

components:
- ../components

commonLabels:
  overlay: this-is-overlay
EOF
```

* 測試：

```bash
kubectl kustomize /tmp/overlays
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    base: this-is-base
    component: this-is-component  
    overlay: this-is-overlay
  name: base-pod
spec:
  containers:
  - image: nginx
    name: my-container
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: this-is-component
    overlay: this-is-overlay
  name: component-pod
spec:
  containers:
  - image: nginx
    name: my-container
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: this-is-component
    overlay: this-is-overlay
  name: overlay-pod
spec:
  containers:
  - image: nginx
    name: my-container
```

從輸出結果可以看到：

* 由於 components 與 overlays 是「先合併後選染」的關係，**overlays 會先將 components 的設定與自己的合併**，再對所有 resource 進行渲染，因此**所有** Pod 都會被貼上 `component: this-is-component` & `overlay: this-is-overlay`  的 Label。

* 由於 base 與 overlays 是「先後渲染」的關係，於是 base 「**先**」對 base-pod 打上 `base: this-is-base` 的 Label，後續 base-pod 被引用到 overlays 時，又因為 components 與 overlays 的「合併」效果，因此額外被貼上 `component: this-is-component` & `overlay: this-is-overlay` 的 Label。


### Component --- patch 模組化

除了能將 Resource 模組化之外，我們也可將 patch 模組化。

什麼意思呢？當有一種 patch 會被重複在不同 overlays 使用時，基於 components 與 overlay 的「合併關係」，就可以先將 patch 寫在 components 中，經由引用將 patch 合併到 overlays，進而對 overlays 的全部資源進行 patch。

舉例來說，只有 test、staging、prod 會將 Deployment 的 replicas 設定為 3，這時就可以將這個 patch 模組化：

* 在 components 目錄下建立一個 patch 子目錄，並撰寫 kustomization.yml & patch.yaml：

```yaml
# components/replicas-patch/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- path: replicas-patch.yaml
  target:
    kind: Deployment
```

```yaml
# components/replicas-patch/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: not-important
spec:
  replicas: 3
```

* 在 test、staging、prod 的 kustomization.yml 引用這個 Component：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

components:
- ../../components/replicas-patch
```

這樣一來，replicas-patch 就可以被重複利用，並且只要更新一次 replicas-patch.yaml，所有引用到的 overlays 都會自動更新。


## 總結

* kustomization.yml 可以互相引用，進行多次的渲染，產生最終的 yaml。

* 根據上述特性，理想的專案目錄架構為：
  * base：放置全環境通用的 yaml 及設定
  * components：將設定模組化，供不同的環境引用
  * overlays：針對不同的環境進行設定

* Base 與 Overlay 的關係是「先後渲染」，Component 與 Overlay 的關係是「先合併後渲染」。

  * 基於「先合併後渲染」的關係，可以將 Resource 與 Patch 模組化，並重複利用。

