# 內建函數

## 計算 counter 增長率

counter 類型只增不減，單單列出這些數值無法看出增長率，這時候可以使用以下方式計算增長率：


* 計算一個 range vector 頭尾的差值

```plaintext
increase(node_cpu_seconds_total{cpu="0",mode="idle"}[2m])
```
> 頭 - 尾

* 計算一個 range vector 的「每秒」的平均增長率：

```plaintext
rate(node_cpu_seconds_total{cpu="0",mode="idle"}[2m])
```
> 頭 - 尾 / 頭尾時間差
or

```plaintext
increase(node_cpu_seconds_total{cpu="0",mode="idle"}[2m]) / 120
```

* rate 因為最後會平均，因此會無法反映出突然的增長，這時候可以使用 `irate`：

```plaintext
irate(node_cpu_seconds_total{cpu="0",mode="idle"}[2m])
```

> irate 使用 range vector 中的「最後兩個數值」計算平均每秒的增長率

**注意**：雖然 `delta` 也可以頭尾的差值，但是 `delta` 不會考慮 counter 重置的情況，例如：

```plaintext
t0: 5
t1: 11
t2: 28
t3: 4
t4: 40
```

* delta 會計算：

```plaintext
40 - 5 = 35
```

* 而 increase 是專門設計給 counter 用的，會考慮 counter 重置的情況：

```plaintext
(28 - 5) + 40 - 0 = 63
```
> 因為 counter 只增不減，所以 4 會被忽略當作 counter 重置為 0。

**因此，delta 適合 gauge，increase 適合 counter。**

## 利用 Gauge 做線性回歸預測：predict_linear()

依據過去的數據，預測未來的數據：

語法：

```plaintext
predict_linear(v range-vector, t scalar)
```

```plaintext
predict_linear(node_filesystem_free_bytes{mountpoint='/var'}[2h], 4 * 3600) 
```
> 根據過去 2 小時的數據，預測未來 4 小時的數據


## 檢查目標是否存活：up{}

up{} 是一種 metric，用來檢查目標是否存活。我們在 {} 加入 label 可篩選出特定目標，若該目標存活，則 up{} 的值為 1，否則為 0。

* 查看所有目標是否存活：

```plaintext
up
```

* 查看 node-exporter 是否存活：

```plaintext
up{job="node-exporter"}
```

## 更換 label：label_replace()

有時候預設的 label 不夠直觀，我們可以透過 `label_replace()` 來更換 label：

```plaintext
label_replace(range-vector, dst_label, replacement, src_label, regex)
```

* 例如 up metric 的 instance 我們只想要其 IP，不想要 port：

```plaintext
label_replace(up, "IP", "$1", "instance", "(.*):.*")
```
> "IP" 可以自行替換成想要的 label 名稱，"$1" 代表 regex 中的第一個 group，也就是 (.*)。