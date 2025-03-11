# Loki Overview

## What is Loki?

Loki 屬於 Grafana Labs 的開發項目，所以能直接支援 Grafana，且比起 EFK、ELK 等其他 log 系統，Loki 更為輕輛，因此適合用在規模較小的 cluster 中。


Loki 只收集 log 的 metadata 並做成 index (timestamp + label)，而非 index 整個 log 的內容，這樣可以節省大量的儲存空間，且查詢時也可以透過 Label，讀取效能更快。

一組擁有相同 Label 的 log 稱為 **log stream**，log data 會被壓縮並儲存，被儲存的 log data 稱為 **chunk**。

> 不同 label 的 log data 所存放的 chunk 也不同。

![alt text](image-1.png)

### Loki logging stack

要完成「Loki 收集 log 的功能 --> Grafana 供使用者查詢」的過程，需要以下幾個元件：

* Agent ：負責收集 log，並將多個 log 加上標籤做成 stream，再把 stream 推送給 Loki。常見的 Agent 如 Grafana Alloy、Promtail。

* Loki ：負責接收 log stream，並將 log data 壓縮儲存到 Object Storage 中。

* Grafana ：提供使用者查詢 log 的介面。使用者可以用 `LogQL` 語言查詢 log。

![alt text](image.png)

https://www.reddit.com/r/grafana/comments/15e9aej/help_setting_up_grafana_prometheus_and_loki_stack/

https://www.civo.com/learn/advanced-kubernetes-monitoring

### Loki architecture

![alt text](image-2.png)

**Write Path**：

1. **Distributor**接收包含 log streams）和 log line 的 HTTP POST 請求。 

2. Distributor 會對請求中的每個 log 流進行 hash運算，以確定應將其發送到哪個 **Ingester**，這是根據 **consistent hash ring** 來決定的。 

3. Distributor 將每個 log stream 發送到適當的 **Ingester**。

4. Ingester 接收 log stream 和 log line，並為該日誌流的數據創建 **chunk**，或將數據追加到現有的 **chunk**。

5. Ingester 確認寫入成功。  

6. Distributor 會等待大多數 **Ingester** 確認其寫入操作。

7. 如果 Distributor 至少收到 N/2 + 1 的確認，則返回 **成功（2xx 狀態碼）**，否則，如果寫入操作失敗，則返回 **錯誤（4xx 或 5xx 狀態碼）**。  


---  

**Read Path**：

1. **Query Frontend** 接收包含 **LogQL 查詢** 的 HTTP GET 請求。

2. Query Frontend 將查詢拆分成 **子查詢（sub-queries）**，並將其傳遞給 **Query Scheduler（查詢調度器）**。  

3. **Querier（查詢器）** 從 **Query Scheduler** 拉取子查詢。  

4. Querier 將查詢傳遞給所有 **Ingester**，以獲取記憶體中的數據。  

5. **Ingester** 返回與查詢匹配的記憶體數據（如果有的話）。  

6. 如果 **Ingester** 沒有 or 返回的數據不足，**Querier** 會從 **後端存儲（backing store）** 延遲加載數據，並對其執行查詢。  

7. Querier 遍歷所有收到的數據，進行 **deduplication**，然後將子查詢結果返回給 **Query Frontend**。  

8. Query Frontend 等待所有子查詢的結果，然後由 **Querier** 返回最終結果。  

9. Query Frontend 合併所有子查詢的結果，生成最終結果並返回給客戶端。  

## PLG stack

若要監控 Kubernetes cluster 的 log，可以使用 **Promtail + Loki + Grafana** 這個組合：

* Promtail：以 DaemonSet 形式部署在每個 Node 上，負責收集 log 並送到 Loki。

* Loki：負責接收 log

* Grafana：提供使用者查詢、觀察 log 的 UI。

### Install Loki stack

```bash

