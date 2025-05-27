# kubectl plugins

> 我們可以透過 `plugin` 來擴充 kubectl 的功能，讓管理 cluster 時更加方便。

## 目錄

* [新增 plugin](#新增-plugin)
  * [自己寫的 plugin](#自己寫的-plugin)
  * [使用 krew 安裝 plugin](#使用-krew-安裝-plugin)
    * [krew 的基本應用](#krew-的基本應用)

* [好用的 plugin list](#好用的-plugin-list)
  * [ns](#ns)
  * [pod-lens](#pod-lens)
  * [iexec](#iexec)
  * [image](#image)
  * [view-allocation](#view-allocation)
  * [sick-pods](#sick-pods)
  * [status](#status)

### 新增 plugin

plugin 說白了就是「執行檔」，一個執行檔要成為 kubectl plugin，必須符合以下規則：

1. 檔名必須以 `kubectl-` 開頭。
2. 執行檔必須在 `$PATH` 中。

> 假設有個名為 `kubectl-foo` 的執行檔，當我們在 terminal 中輸入「kubectl foo」時，kubectl 會自動呼叫 `kubectl-foo` 執行檔來執行。

Plugin 執行檔可以是任何語言寫的，無論是 shell script、Python、Go 都可以。一般來說，plugin 的來源可分為兩種：

1. 自己寫的 plugin。

2. 使用 `krew` 安裝的 plugin。

### 自己寫的 plugin

假設今天有一個 plugin 叫做 `kubectl-hello`，它的功能是輸出「Hello, Kubernetes!」：


* 撰寫一個名為 `kubectl-hello` 的 shell script，內容如下：
  
  ```bash
  #!/bin/bash
  # kubectl-hello
  
  # shows plugin version
  if [[ "$1" == "version" ]]; then
    echo "kubectl-hello v1.0.0"
    exit 0
  fi
  
  # shows plugin help
  
  if [[ "$1" == "help" ]]; then
    echo "Usage: kubectl hello"
    echo "A simple plugin that says hello to Kubernetes."
    exit 0
  fi
  
  echo "Hello, Kubernetes!"
  ```

* 給予執行權限後，將它放到 `$PATH` 中的某個目錄，例如 `/usr/local/bin`：

  ```bash
  chmod +x kubectl-hello
  sudo mv kubectl-hello /usr/local/bin/
  ```

現在我們就可以在 terminal 中輸入 `kubectl hello` 來執行這個 plugin：

```bash
kubectl hello
# Hello, Kubernetes!

kubectl hello version
# kubectl-hello v1.0.0

kubectl hello help
# Usage: kubectl hello
# A simple plugin that says hello to Kubernetes.
```

### 使用 krew 安裝 plugin

[krew](https://krew.sigs.k8s.io/) 是 kubectl 的 plugin 管理工具，可以讓我們輕鬆地安裝、更新和管理 kubectl plugins。

> 可以理解成 kubectl plugin 的 apt/yum。

#### 安裝 krew

在 Linux 或 macOS 上安裝 krew 非常簡單，只需要執行以下命令：

> 注意：先確任系統中已安裝 `git`，再安裝 krew。

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

安裝後，依提示將新的 `$PATH` 寫入 .bashrc 或 .zshrc：

```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

執行以下命令使改動生效：

```bash
source ~/.bashrc
```

最後，檢查 krew 是否安裝成功：

```bash
kubectl krew 
```
> 如果看到 krew 的使用說明，表示安裝成功。

#### krew 的基本應用

* 更新可用的 plugin 列表：

  ```bash
  kubectl krew update
  ```

* 列出所有可下載的 plugin:

  ```bash
  kubectl krew search
  ```

* 列出 plugin 的詳細資訊：

  ```bash
  kubectl krew info <plugin-name>
  ```

* 安裝 plugin：

  ```bash
    kubectl krew install <plugin-name>
  ```

* 列出已安裝的 plugin：

  ```bash
  kubectl krew list
  ```

* 卸載 plugin：

  ```bash
  kubectl krew uninstall <plugin-name>
  ```

* 更新已安裝的 plugin：

  ```bash
  kubectl krew upgrade
  ```

* 查看 plugin 的使用說明：

  ```bash
  kubectl <plugin-name> --help
  ```

### 好用的 plugin list

這裡整理一些推薦使用的 kubectl plugin，皆可以透過 `krew` 安裝。

#### ns

以前如果常常在不同 namespace 中操作 kubectl，可能會覺得每次都要加上 `-n <namespace>` 很麻煩。這時候可以使用 `ns` plugin 來簡化操作。

* 列出所有 namespace：

  ```bash
  kubectl ns
  ```

* 切換 namespace：

  ```bash
    kubectl ns <namespace>
  ```

* 列出當前 namespace：

  ```bash
  kubectl ns -c
  ```

* 切換回上一個 namespace：

  ```bash
  kubectl ns -
  ```
  > 類似 `cd -` 的功能。 


#### pod-lens

一次列出 Pod 的詳細資訊，包括 Pod 的狀態、IP、Node、容器資訊、configmap 等等。

* 互動式的挑選當前 namespace 的 Pod：

  ```bash
  kubectl pod-lens
  ```

* 篩選出名字中帶有某字串的 Pod：

  ```bash
  kubectl pod-lens <keyword>
  ```

* 使用 label 篩選 Pod:

  ```bash
  kubectl pod-lens -l <label-selector>
  ```

* 查看某個 pod 的詳細資訊：

  ```bash
  kubectl pod-lens <pod-name>
  ```

#### iexec

其實就是簡化版的 `kubectl exec`，可以更快速的在 Pod 中執行命令。

* 讓使用者在當前 namespace 中選擇 Pod，然後開啟一個 shell 與 Pod 互動：
  ```bash
  kubectl iexec
  ```

* 指定 Pod 名稱，直接開啟一個 shell 與 Pod 互動：

  ```bash
  kubectl iexec <pod-name>
  ```

* 在 Pod 中執行命令：

  ```bash
  kubectl iexec <pod-name> <command>
  ```

#### image

列出 image 相關的資訊，例如 image 正在被哪些 Pod 使用、image 的大小、所屬 repo、tag、sha 等等。

* 列出當前 namespace 中所有 Pod 使用的 image：

  ```bash
  kubectl image list
  ```

* 列出**所有** Pod 使用的 image：

  ```bash
  kubectl image list --all-namespaces
  ```

* 更詳細的列出 image 的資訊：

  ```bash
  kubectl image get
  ```

#### view-allocation

如果 Pod 有設定資源限制（requests 和 limits），可以使用 `view-allocation` plugin 來查看 Pod 的資源分配情況。

> 沒有設定 requests 和 limits 的 Pod 則不會顯示。

```bash
kubectl view-allocation
```

#### sick-pods

列出那些狀態為 "NotReady" 的 Pod，並顯示其詳細資訊。

* 列出當前 namespace 中所有狀態為 "NotReady" 的 Pod：

  ```bash
  kubectl sick-pods
  ```
  ```text
  'nginx' is not ready! Reason Provided: None
        Failed Pod Conditions:
                CONDITION       REASON                  MESSAGE
                Ready           ContainersNotReady      containers with unready status: [nginx]
                ContainersReady ContainersNotReady      containers with unready status: [nginx]

        Pod Events:
                LAST SEEN                       TYPE    REASON          MESSAGE
                2025-05-27 16:55:40 +0800 CST   Normal  Scheduled       Successfully assigned default/nginx to worker-5
                2025-05-27 16:55:40 +0800 CST   Normal  Pulling         Pulling image "nginx"
                2025-05-27 16:55:54 +0800 CST   Normal  Pulled          Successfully pulled image "nginx" in 13.545s (13.545s including waiting). Image size: 72402122 bytes.
                2025-05-27 16:55:54 +0800 CST   Normal  Created         Created container: nginx
                2025-05-27 16:55:54 +0800 CST   Normal  Started         Started container nginx
                2025-05-27 17:21:29 +0800 CST   Normal  Killing         Stopping container nginx
                2025-05-27 17:21:53 +0800 CST   Normal  Scheduled       Successfully assigned default/nginx to worker-5
                2025-05-27 17:22:08 +0800 CST   Normal  Pulling         Pulling image "nginxx"
                2025-05-27 17:22:09 +0800 CST   Warning Failed          Failed to pull image "nginxx": failed to pull and unpack image "docker.io/library/nginxx:latest": failed to resolve reference "docker.io/library/nginxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
                2025-05-27 17:22:09 +0800 CST   Warning Failed          Error: ErrImagePull
                2025-05-27 17:21:55 +0800 CST   Normal  BackOff         Back-off pulling image "nginxx"
                2025-05-27 17:21:55 +0800 CST   Warning Failed          Error: ImagePullBackOff

        Container 'nginx' is not ready!
                Container Logs:
                        Errored Getting Logs: streaming log results: container "nginx" in pod "nginx" is waiting to start: trying and failing to pull image
    ```
    > 如輸出結果所示，有個叫做 `nginx` 的 Pod 狀態為 "NotReady"，原因是 image 拉取失敗。

#### status

將 Pod 的狀態以及相關事件整理出來，在 debug 時非常有用。

```bash
kubectl status pods nginx
```
```text
Pod/nginx -n default, created 3m ago, gen:1 Pending BestEffort
  InProgress: Pod is in the Pending phase
    Reconciling: PodPending, Pod is in the Pending phase
  PodScheduled -> Initialized -> Not ContainersReady -> Not Ready
    Ready ContainersNotReady, containers with unready status: [nginx] for 3m
    ContainersReady ContainersNotReady, containers with unready status: [nginx] for 3m
  Standalone POD.
  Containers:
    nginx (nginxx) Waiting ImagePullBackOff: Back-off pulling image "nginxx": ErrImagePull: failed to pull and unpack image "docker.io/library/nginxx:latest": failed to resolve reference "docker.io/library/nginxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Known/recorded manage events:
    3m ago Updated by kubectl-run (metadata, spec)
    3m ago Updated by calico (metadata)
    25s ago Updated by kubelet (status)
  Events:
    Scheduled 3m ago from default-scheduler,default-scheduler: Successfully assigned default/nginx to worker-5
    Pulling 52s ago (x5 over 3m) from kubelet,worker-5,kubelet,worker-5: Pulling image "nginxx"
    Failed 51s ago (x5 over 3m) from kubelet,worker-5,kubelet,worker-5: Failed to pull image "nginxx": failed to pull and unpack image "docker.io/library/nginxx:latest": failed to resolve reference "docker.io/library/nginxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
    Failed 51s ago (x5 over 3m) from kubelet,worker-5,kubelet,worker-5: Error: ErrImagePull
    BackOff 13s ago (x13 over 3m) from kubelet,worker-5,kubelet,worker-5: Back-off pulling image "nginxx"
    Failed 13s ago (x13 over 3m) from kubelet,worker-5,kubelet,worker-5: Error: ImagePullBackOffW0527 09:25:41.733302  488249 warnings.go:70] v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
  ```
  > 可以看到，nginx Pod 的經歷了 `PodScheduled -> Initialized -> Not ContainersReady -> Not Ready`，且不屬於任何 Deployment、StatefulSet (Standalone POD)，無法啟動的原因是 image 拉取失敗，並且有相關的事件記錄。




