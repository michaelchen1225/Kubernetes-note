# Data Structure：time series

Prometheus 抓到的數據會以 time series 的形式保存，每個 time series 由 metric name、labels、timestamp、value 組成，並依照 timestamp 排序。

```plaintext
  ^
  │   . . . . . . . . . . . . . . . . .   . .   node_cpu{cpu="cpu0",mode="idle"}
  │     . . . . . . . . . . . . . . . . . . .   node_cpu{cpu="cpu0",mode="system"}
  │     . . . . . . . . . .   . . . . . . . .   node_load1{}
  │     . . . . . . . . . . . . . . . .   . .  
  v
    <------------------ time ---------------->
```

time series 的結構：

```plaintext
<--------------- metric ---------------------><-timestamp -><-value->
http_request_total{status="200", method="GET"}@1434417560938 => 94355
http_request_total{status="200", method="GET"}@1434417561287 => 94334

http_request_total{status="404", method="GET"}@1434417560938 => 38473
http_request_total{status="404", method="GET"}@1434417561287 => 38544

http_request_total{status="200", method="POST"}@1434417560938 => 4748
http_request_total{status="200", method="POST"}@1434417561287 => 4785
```

> 有一種 metric 為 http_request_total，使用者可以透過 label 來篩選出特定的數據，例如 status=200|404、method=GET|POST。

## Metric types

* Counter

    * Value 只能增加，不能減少。可被重設為 0。
    * 通常後綴為 `_total`
    * ex. request 總數

* Gauge

    * Value 可以增加或減少
    * ex. pod 個數

* Histogram
    
    * 計算數據的分佈(return: buckets, sum, count)
    * 可用 `histogram_quantile()` 計算百分位數
    * ex. prometheus_tsdb_compaction_chunk_range_bucket

```plaintext
# HELP prometheus_tsdb_compaction_chunk_range Final time range of chunks on their first compaction
# TYPE prometheus_tsdb_compaction_chunk_range histogram
prometheus_tsdb_compaction_chunk_range_bucket{le="100"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="6400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="25600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="102400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="409600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1.6384e+06"} 260
prometheus_tsdb_compaction_chunk_range_bucket{le="6.5536e+06"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="2.62144e+07"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="+Inf"} 780
prometheus_tsdb_compaction_chunk_range_sum 1.1540798e+09
prometheus_tsdb_compaction_chunk_range_count 780
```

> 總共有 780 個 chunks，其中 260 個 chunks 落在 1.6384e+06 以內(less than or equal)。

* Summary

    * 類似 histogram，但是返回的是 quantiles(百分位數)
    * ex. prometheus_tsdb_compaction_chunk_range

```plaintext
# HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
# TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216
```

> 三個百分位數：50% 的 fsync 操作在 0.012352463 秒以內，90% 在 0.014458005 秒以內，99% 在 0.017316173 秒以內。共有 216 次 fsync 操作、共耗時 2.888716127 秒。

https://prometheus.io/docs/tutorials/understanding_metric_types/


https://ithelp.ithome.com.tw/m/articles/10322080

https://yunlzheng.gitbook.io/prometheus-book

## 選擇合適的 metric

若用平均數來衡量，可能會受到極端值的影響，因此可以使用 quantile 來衡量。

例如：平均而言，一個 request 的處理時間為 1 秒，但是有 1% 的 request 處理時間為 10 秒，使用平均數來衡量會失真，因此可以使用 99% 的 quantile 來衡量。