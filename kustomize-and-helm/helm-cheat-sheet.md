# Helm Cheat Sheet

## 目錄

* [前情提要](#前情提要)

* [初始化 Helm Chart 目錄](#初始化-helm-chart-目錄)

* [檢查 Chart](#檢查-chart)

* [輸出渲染後的 template](#輸出渲染後的-template)

* [安裝 chart](#安裝-chart)

* [安裝 Chart 並帶入新的 value.yaml](#安裝-chart-並帶入新的-valueyaml)

* [把 Chart 安裝在一個全新的 namespace](#把-chart-安裝在一個全新的-namespace)

* [安裝 Chart 並直接指定 value](#安裝-chart-並直接指定-value)

* [檢查 Chart 是否能安裝成功](#檢查-chart-是否能安裝成功)

* [列出某個 namespace 中的 release](#列出某個-namespace-中的-release)

* [列出所有 namespace 中的 release](#列出所有-namespace-中的-release)

* [解除安裝 Chart](#解除安裝-chart)

* [列出已安裝的 release](#列出已安裝的-release)

* [更新一個 release](#更新一個-release)

* [查看某個 release 的版本紀錄](#查看某個-release-的版本紀錄)

* [Rollback 一個 release 到指定 REVISION](#rollback-一個-release-到指定-revision)

* [檢查兩次 REVISION 的差異](#檢查兩次-revision-的差異)

* [打包 Chart 成一個 tgz 檔](#打包-chart-成一個-tgz-檔)

* [Remote repo 相關](#remote-repo-相關)

  * [加入遠端 repo](#加入遠端-repo)
  
  * [更新所有 repo](#更新所有-repo)

  * [列出目前可用的 repo](#列出目前可用的-repo)

  * [移除 repo](#移除-repo)

  * [在已經加入的 repository 中以關鍵字搜尋 Chart](#在已經加入的-repository-中以關鍵字搜尋-chart)

  * [列出某個 Chart 可用的所有版本](#列出某個-chart-可用的所有版本)

  * [安裝遠端 repo 的 Chart](#安裝遠端-repo-的-chart)

  * [安裝遠端 repo 的 Chart 並指定版本](#安裝遠端-repo-的-chart-並指定版本)

  * [取得 Chart 的 value.yaml](#取得-chart-的-valueyaml)

  * [更新一個 release，且 chart 來源為遠端 repo](#更新一個-release且-chart-來源為遠端-repo)





## 前情提要

* 不同 namespace 的操作請用 `-n` 指定，以下預設為 default namespace。

* 下方說明中的 `<chart-name>` 在指令中的表達方式可依需求自行選擇：

    1. `/path/to/chart-name/`：本地的 chart 路徑

    2. `<repo-name>/<chart-name>`：遠端 repo 中的某個 Chart

    3. `chart-name.tgz`：本地的 Chart 打包檔

   例如「helm template \<chart-name>」可以是：

    ```bash
    helm template /path/to/chart-name/
    ```
    ```bash
    helm template <repo-name>/<chart-name>
    ```
    ```bash
    helm template chart-name.tgz
    ```

### 初始化 Helm Chart 目錄

> 初始化一個 Chart，生成基本的目錄結構：

```bash
helm create <chart-name>
```

### 檢查 Chart

```bash
helm lint <chart-name>
```

### 輸出渲染後的 template

> 通常用來驗證是否符合預期：

```bash
helm template <chart-name> -f <values.yaml>
```

### 安裝 chart

```bash
helm install <release-name> <chart-name>
```

### 安裝 Chart 並帶入新的 value.yaml 

```bash
helm install <release-name> <chart-name> --values </path/to/other/values.yaml>
```

or

```bash
helm install <release-name> <chart-name> -f </path/to/other/values.yaml>
```

### 把 Chart 安裝在一個全新的 namespace
```bash
helm install <release-name> <chart-name> -n <namespace> ----create-namespace
```

### 安裝 Chart 並直接指定 value

```bash
helm install <release-name> <chart-name> --set <key>=<value>
```

### 檢查 Chart 是否能安裝成功

> 僅檢查安裝 Chart 是否能成功，但並不是真的安裝：

```bash
helm install <release-name> <chart-name> --dry-run
```

### 列出某個 namespace 中的 release
```bash
helm list -n <namespace>
```

### 列出所有 namespace 中的 release
```bash
helm list --all-namespaces
```

### 解除安裝 Chart
```bash
helm uninstall <release-name>
```

### 列出已安裝的 release
```bash
helm list
```

### 更新一個 release
```bash
helm upgrade <release-name> <chart-name>
```

### 查看某個 release 的版本紀錄

```bash
helm history <release-name>
```

### Rollback 一個 release 到指定 REVISION

```bash
helm rollback <release-name> <revision>
```

### 檢查兩次 REVISION 的差異

```bash
helm diff revision <release-name> <revision-1> <revision-2>
```

### 打包 Chart 成一個 tgz 檔

```bash
helm package </path/to/chart-name/>
```

## Remote repo 相關

### 加入遠端 repo

```bash
helm repo add <repo-name> <repo-url>
```

### 更新所有 repo

```bash
helm repo udpate
```

### 列出目前可用的 repo

```bash
helm repo list
```

### 移除 repo

```bash
helm repo remove <repo-name>
```

### 在已經加入的 repository 中以關鍵字搜尋 Chart

```bash
helm search repo <key-words>
```

### 列出某個 Chart 可用的所有版本

```bash
helm search repo <repo-name>/<chart-name> --versions
```

### 安裝遠端 repo 的 Chart

```bash
helm install <release-name> <repo-name>/<chart-name>
```
### 安裝遠端 repo 的 Chart 並指定版本

```bash
helm install <release-name> <repo-name>/<chart-name> --version <version-number>
```

### 取得 Chart 的 value.yaml

```bash
helm show values <repo-name>/<chart-name>
```


### 更新一個 release，且 chart 來源為遠端 repo：
```bash
helm repo update
helm upgrade <release-name> <repo-name>/<chart-name>
```


