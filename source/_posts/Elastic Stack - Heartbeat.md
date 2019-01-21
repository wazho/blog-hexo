---
title: Elastic Stack - Heartbeat
date: 2018-11-14 20:00:00
---

## 安裝

[Elastic Heartbeat guide](https://www.elastic.co/guide/en/beats/heartbeat/current/running-on-docker.html)

### A. 前置作業

1. 準備 Elasticsearch host，版本建議與 heartbeat 相同
2. Elasticsearch 需要註冊 licence
    * 若版本為 6.3.x 以上：https://www.elastic.co/guide/en/elasticsearch/reference/6.3/start-basic.html
    * 若版本為 6.2.x 以下：https://register.elastic.co/marvel_register


### B. 步驟流程

#### 1. 下載 docker image

```shell
docker pull docker.elastic.co/beats/heartbeat:6.4.3 
```

#### 2. 建立 heartbeat.yml

**heartbeat.yml** 為 Heartbeat 的設定檔，當中定義了欲監控的服務，與輸出資料的寫入對象等設定。

關於 **setup.template** 的說明，請參考 [Load the index template in Elasticsearch](https://www.elastic.co/guide/en/beats/heartbeat/current/heartbeat-template.html#heartbeat-template)。

> 該檔案若有任何異動，Heartbeat 服務則需要重新啟動。

```yaml
setup.template.name: "my-heartbeat"
setup.template.pattern: "my-heartbeat-*"
# 預期要監控的 serives
heartbeat.monitors:
  - type: http
    # 亦支援 cron-like syntax
    schedule: '@every 5s'
    urls: ["http://my-domain/api/"]
    check.response.status: 200
heartbeat.scheduler:
  limit: 10
# 指定 ES 作為寫入對象
output.elasticsearch:
  hosts: ["http://my-es-host:9200"]
  index: "heartbeat-%{[beat.version]}-%{+yyyy.MM.dd}"
# 啟用 kibana 作為 dashboard
setup.kibana:
  host: "my-kibana-host:5601"
```

#### 3. 運行 docker container

```shell
docker run -d \
    --mount type=bind,source="$(pwd)"/heartbeat.yml,target=/usr/share/heartbeat/heartbeat.yml \
    --name heartbeat \
    docker.elastic.co/beats/heartbeat:6.4.3
```



## 運行方式

### 週期性呼叫 API

Heartbeat 根據設定檔的 **heartbeat.monitors** 進行監控。

在官方文件 [Exported fields][heartbeat-exported-fields] 定義了能夠輸出的資訊，其中以下欄位是默認輸出：
* [Host fields][exported-fields-host-processor]
* [TLS encryption layer fields][exported-fields-tls]
* [HTTP monitor fields][exported-fields-http]
* [Common heartbeat monitor fields][exported-fields-common]
* [Host lookup fields][exported-fields-resolve]
* [TCP layer fields][exported-fields-tcp]
* [Beat fields][exported-fields-beat] (涵蓋 error fields)


[heartbeat-exported-fields]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields.html
[exported-fields-host-processor]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-host-processor.html
[exported-fields-tls]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-tls.html
[exported-fields-http]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-http.html
[exported-fields-common]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-common.html
[exported-fields-resolve]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-resolve.html
[exported-fields-tcp]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-tcp.html
[exported-fields-beat]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields-beat.html

### 發送至 ES 儲存數據

根據設定檔 **output.elasticsearch.index** 定義之 pattern 作為 index 的命名規則。


### 透過 Kibana 呈現數據

根據設定檔 **setup.kibana** 配置，當 Heartbeat 啟動時會建立 Heartbeat visualize & dashboard。

![](//hackmd.webpat.co/uploads/upload_f932e3ce64539196bb3e17ee7418d431.jpg)


## 備註

### 若憑證失效

當 Heartbeat 發送紀錄至 Elasticsearch 時，Heartbeat 會因為該 ES 的憑證過期出現以下錯誤訊息：

```shell
[ERROR][o.e.x.s.a.f.SecurityActionFilter] [mpzf-n0] blocking [cluster:monitor/stats] operation due to expired license. Cluster health, cluster stats and indices stats ,
operations are blocked on license expiration. All data operations (read and write) continue to work. ,
If you have a new license, please update it. Otherwise, please reach out to your support contact.,
```
