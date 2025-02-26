# Prometheus

## Prometheus 是什麼？

首先得先提到 metric 的概念。所謂 metric 其實就是某種指標，例如 CPU 使用率、記憶體使用率、API 請求次數等等。適當的收集有用的 metric，對系統維運與除錯相當有幫助。

而 Prometheus 就是一套開源的 metric 收集系統，使用 HTTP 週期性的抓取 metric，將 metric 儲存在 time-series database 中，並提供一個 API 讓我們調用、查詢過去的紀錄(可以拿來做圖表、dashboard)。

> TSDB (Time Series Database) 專門設計用於高效儲存和處理**時間序列資料**的資料庫系統。

## Prometheus 的架構

以下為一個常見的 Prometheus + Grafana 的應用場景：

![alt text](image.png)

Prometheus 的架構主要分為以下幾個部分：

* **Prometheus Server**：
  * Retrival：週期性的抓取 metric。
  * Storage：將 metric 儲存在 time-series database 中。
  * HTTP Server：提供一個 API，他人需使用「PromQL」調用、查詢過去的紀錄。

* **Alertmanager**：
  * 負責接收 Prometheus Server 發出的警告訊息，並根據設定的規則進行警報通知。

* **exporter**：
  * 用來將不同的 metric 轉換成 Prometheus Server 可以提取的格式。
    * 例如：Node exporter、Kube-state-metrics、Kubernetes control plane metrics。

* **Library**：
  * Prometheus 提供了不同程式語言的 Library，可以用來將程式執行中的 metric 輸出。

* **Pushgateway**：
  * 假設我們自己寫了一個小腳本來抓取 metric，可以丟到這個 Pushgateway，再交由 Prometheus Server 提取。

## Prometheus 如何設定 target？

安裝好 Prometheus Server 之後，我們需要告訴 Prometheus Server 要去哪裡抓取 metric，可透過 yaml 檔來指定。例如：

```yaml
global:
  scrape_interval: 15s # 每 15 秒抓取一次 metric

scrape_configs:
  - job_name: prometheus # 自行命名 
    static_configs:
      - targets: ["localhost:9090"] # 目標的位置
```

> 上述 yaml 檔告訴 Prometheus Server 每 15 秒抓取一次 localhost:9090 的 metric(這邊是 Prometheus Server 自己)。

因為 Prometheus Server 本身就提供 metric 可供抓取，所以不需額外安裝甚麼。但假如今天想要抓取 Server 上的 metric，就需要安裝相對應的 exporter 在 target 上。 

假設安裝好了 node exporter，就可以透過以下 yaml 檔告訴 Prometheus Server 去哪裡抓取 metric：

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: node_exporter
    static_configs:
      - targets: ["localhost:9100"]
```

## Prometheus 的 data 格式

收集的數據被存在 time-series database 中，當然重點就是「時間」，那該如何區分不同 metric 的時間呢？可以透過「命名 metric」的方式，例如：

```bash
# label 可以有多個
<metric name>{<label name>=<label value>, ...}
```

metric name + label 就是一個 metric，在資料庫中，每個 metric 後面會加上 float64 的浮點數(代表當前樣本的數值)：

```
http_requests_total{method="POST", handler="/messages"} 20 
```
> 代表在時間點 t，對 /messages 發送 POST 請求的次數總共為 20 次


## PromQL 簡介

PromQL 是 Prometheus 的查詢語言，提供了一系列的函數可以用來查詢、過濾、計算 metric，例如：

* 調取 metric name = http_requests_total 底下所有的時間序列：

```sql
http_requests_total
```

* 調取所有 metric name 底下 label 為 method = "POST" 的時間序列：

```sql
http_requests_total{method="POST"}
```

* 調取過取 5 分鐘內的時間序列：

```sql
http_requests_total{method="POST"}[5m]
```

* 可用函數的範例：[網址](https://promlabs.com/promql-cheat-sheet/)：

## kube prometheus stack

當我們在想透過 Prometheus 監控 k8s 時，通常還會搭配其他工具，如果自己安裝會裝很久，kube prometheus stack 就是一個將一堆工具包在一起的 helm chart，其中整合了：

* Prometheus

* Alertmanager

* Grafana：dashboard 視覺化工具

* node-exporter：收集 node 上的相關 metric

* kube-state-metrics：透過 kube-apiserver 收集 k8s cluster 的相關 metric(例如：deployment、pod、service 等等)

* Prometheus Operator：用於在 Kubernetes 中自動化 Prometheus 和 AlertManager 的配置和運維，並且能讓使用者自行調整相關設定。





