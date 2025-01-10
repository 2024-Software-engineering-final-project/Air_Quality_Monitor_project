### 安裝說明
+ 在[官網下載](https://prometheus.io/download/)

### 使用說明
請將alerts.yml 放入 Prometheus.exe 相同路徑中
並將 更新後的 prometheus.yml 更新至 server 上
最後 將alertmanager 那包也放上伺服器並執行

附註(更新 prometheus.yml 的部分)：只有更新alerts 那邊。加入 alertmanager 的預設位置，讓 prometheus知道要去哪接收 alert.

```
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - "140.123.101.199:9093"
```