# Helm vs Kustomize

* helm 不只能針對環境客製化 yaml，他也是一個 package manager，可以將 yaml 打包成一個 chart，並且可以透過 `helm install` 來部署。

* helm 使用 go template syntax 來客製化 yaml。(kustomizs 純 yaml 比較好讀)

I'd say that for internal use only Kustomize is your friend because of greater (imo) readability and with internal tools you don't always need that much customization that helm provides. My team uses helm beacuse we also distribute the software to clients if they want to setup the applications themselves + there are only two people in my team that are kubernetes oriented (except from me) and they feel better with helm