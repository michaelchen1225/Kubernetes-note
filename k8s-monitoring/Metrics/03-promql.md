# PromQL

## 目錄

* [Data type](#data-type)

* [查詢時間序列 (Instant vector)](#查詢時間序列-instant-vector)

* [範圍查詢 (Range vector)](#範圍查詢-range-vector)

* [時間位移：offset](#時間位移offset)

* [聚合操作](#聚合操作)

* [運算](#運算)

* [bool 運算](#bool-運算)

## Data type

* Instant vector：一個時間點的數據

* Range vector：一段時間內的數據

* Scalar：單純的浮點數 (不含時間資訊)

* String：字串 (不含時間資訊)


## 查詢時間序列 (Instant vector)

* 直接查詢 metric 的所有時間序列：

```plaintext
node_cpu_seconds_total
```

* 查詢特定 label 的時間序列：

```plaintext
node_cpu_seconds_total{cpu="0",mode="idle"}
```

* 排除特定 label 的時間序列：

```plaintext
node_cpu_seconds_total {cpu!="0"}
```

* 使用正規表達式：

```
node_cpu_seconds_total{cpu=~"0|1"}
```

or

```plaintext
node_cpu_seconds_total{cpu!~"0|1"}
```

## 範圍查詢 (Range vector)


時間單位：

* `s` 秒

* `m` 分鐘

* `h` 小時

* `d` 天

* `w` 週

* `y` 年


* 查詢過去 5 分鐘某 metric 的時間序列：

```plaintext
node_cpu_seconds_total[5m]
```

## 時間位移：offset

* 查詢過去 5 分鐘某 metric 的 Instant vector：

```plaintext
node_cpu_seconds_total offset 5m
```

## 聚合操作

聚合函數：

* sum（求和）

* min（最小值）

* max（最大值）

* avg（平均值）

* stddev（標準差）

* stdvar（標準方差）

* count（計數）

* count_values（對值進行計數）

* bottomk（後 n 條時序）

* topk（前 n 條時序）

* quantile（分位數）

語法：

```plaintext
<aggr-op>(<vector-expression>) without|by (<label list>)
```

* 求和：

```plaintext
sum(node_cpu_seconds_total)
```

* 將不同 mode 分組後，再列出各組的總和

```plaintext
sum(node_cpu_seconds_total) by (mode)
```

* 依照 label 分類，但排除 mode 這個 label

```plaintext
sum(node_cpu_seconds_total) without (mode)
```

* 計算 mode = idle 的 metric 個數：

```plaintext
count(node_cpu_seconds_total{mode="idle"})
```

* 依照 mode 與 namespace 分組，算出每組有多少個 metric：

```plaintext
count(node_cpu_seconds_total) by (mode, namespace)
```

* 計算「值」的數量：

```plaintext
count_values("value", node_cpu_seconds_total)
```
> "value" 名稱可以自訂，上述 query 會回傳一個新的 time series，且帶有一個新的 label "value"

* 取得 value 前 5 高的 time series：

```plaintext
topk(5, node_cpu_seconds_total)
```

* 取得 value 最低的 5 個 time series：

```plaintext
bottomk(5, node_cpu_seconds_total)
```

* 計算中位數：

```plaintext
quantile(0.5, node_cpu_seconds_total)
```

## 運算

* `+`

* `-`

* `*`

* `/`

* `%

例如：將一個時間序列除以 1024：

```plaintext
node_memory_MemFree_bytes / 1024
```
> Instant vector / scalar

## bool 運算

* `==`

* `!=`

* `>`

* `<`

* `>=`

* `<=`

ex. 找出 memory 使用率超過 80% 的 node：

```plaintext
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes) / node_memory_MemTotal_bytes > 0.8
```

