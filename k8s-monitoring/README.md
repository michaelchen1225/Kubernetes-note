# Monitoring

> [章節列表](#章節列表)

> [安裝 Monitoring tools：Prometheus + Grafana + Loki](./log/01-install-loki.md)

## Three Observability Pillars

* **Metrics**：指標，系統的量化指標，如：CPU 使用率、API 回應時間

* **Logs**：日誌，系統中發生的事情的紀錄，如：錯誤訊息、Exception

* **Traces**：鏈路追蹤，Request 在不同服務中的歷程，例如在 iThome 使用 Facebook 的 SSO（Single Sign-On），一個單純的登入 “Request”，可能就橫跨了 iThome 的 Backend Service、Facebook 的 SSO Backend、iThome 的 Redis，這些衍生出來的 Requests，跟最初的登入 Request 結合起來呈現的歷程就稱為 Traces。

## 資料收集 Pattern

* **Push**：生成資訊的服務透過推送的方式把資料傳遞出去，收集資訊的服務被動收集資訊，常見搭配的動詞有 Push 或 Write。

* **Pull**：生成資訊的服務揭露資訊後被動地等待別人來取用，收集資訊的服務主動去收集資訊，常見搭配的動詞有 Collect 或 Scrape。

* **Proxy**：介於生成資訊的服務與收集資訊的服務之間，可能主動或被動收集資訊後再將資訊轉發給收集資訊的服務，常見搭配的動詞有 Aggregate。

## 章節列表

### Metrics

* [Prometheus 簡介](./Metrics/00-prometheus.md)

* [Prometheus Data structure](./Metrics/01-data-structure.md)

* [PromQL](./Metrics/03-promql.md)

* [PromQL Functions](./Metrics/04-function.md)


### Logs

* [Loki 簡介](./log/00-overview-loki.md)

* [安裝 Loki](./log/01-install-loki.md)