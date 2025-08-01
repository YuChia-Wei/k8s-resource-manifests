# kubernetes resource manifests

This repository stores the resource files required by a Kubernetes cluster that are not packaged by Helm as kustomizations. In conjunction with ArgoCD, these resources can be quickly deployed to a Kubernetes cluster.

## RebbitMQ

此份 k8s 資源目錄所使用的 RebbitMQ 倚賴 RebbitMQ Operator，利用 ArgoCD 安裝時應以以下順序進行同步、安裝

1. rebbitmq-operator
    > 這邊會附帶同步(安裝)PodMonitor，如果還沒安裝 prometheus 的話，會出現 PodMonitor 同步失敗的錯誤
2. rebbitmq-cluster

rebbitmq-grafana-dashboards 的部分則依據需求進行安裝

### version

| app               | version          | release url                                                     |
|-------------------|------------------|-----------------------------------------------------------------|
| rebbitmq-operator | 2.15.0           | [2.15.0](https://github.com/rabbitmq/cluster-operator/releases) |
| rebbitmq-cluster  | 4.1.2-management |                                                                 |

rebbitmq-cluster 的 image 來源有分 `rabbitmq:<version>-management` 與 `rabbitmq:<version>` 兩種，前者預設開啟 UI 與觀測、套件，後者則需要自行設定。

#### rebbit 映像檔差異

> by AI

在官方 RabbitMQ Docker Hub 會看到兩條主要的標籤路線：

| 範例標籤                                                       | 特色                                                                                                                                                                                                                                                                                     | 預設開放埠                                                                         |
|----------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| `rabbitmq:3.13` （或 `rabbitmq:<版本>`）                       | **純粹的 Broker**：只有核心 RabbitMQ 服務，未啟用任何進階外掛。                                                                                                                                                                                                                          | 5672（AMQP）、25672（Erlang 分散式通訊）                                           |
| `rabbitmq:3.13-management` （或 `rabbitmq:<版本>-management`） | **已啟用 Management Plugin**：在映像建置過程中執行 `rabbitmq-plugins enable --offline rabbitmq_management`（以及預設一起啟用的 `rabbitmq_prometheus`）<br> → 內建瀏覽器版管理介面與 HTTP API，可即時檢視/操作佇列、交換器、連線、節點狀態…<br> → 額外開啟 Prometheus 指標端點 `/metrics` | 5672、25672 **外加** 15672（HTTP UI）、15671（HTTPS UI）、15692（Prometheus 指標） |

> 官方文件敘述：「There is a second set of tags provided with the **management plugin installed and enabled by default**…」 ([hub.docker.com][1])
> Stack Overflow 上也點出 *management* 版其實就是「`rabbitmq` 基底映像再多做 `rabbitmq-plugins enable rabbitmq_management`」而已。 ([Stack Overflow][2])

##### 實際影響

1. **功能**

   * **純版號**：體積較小；若後續需要管理 UI，得在執行中手動 `rabbitmq-plugins enable rabbitmq_management`，並自行開放 15672/15671。
   * **-management**：開箱即用的 Web UI、HTTP API、Prometheus 指標，不必再手動啟用外掛。

2. **映像大小**
   management 版因多了 `rabbitmq_management`、`cowboy`、`prometheus` 等套件，大約比純版號大 5–10 MB 左右（依版本差異不大）。

3. **網路安全面**
   若只當作內部訊息代理、不需要遠端管理 UI，建議使用純版號並關閉 1567x 埠，可減少潛在攻擊面。

##### 快速示範

```bash
# 純 Broker（需要時再手動啟用 plugin）
docker run -d --name mq \
  -p 5672:5672 rabbitmq:3.13

# 立即可用的管理 UI
docker run -d --name mq-mgmt \
  -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management
# 瀏覽 http://localhost:15672  預設帳密 guest/guest
```

##### 什麼時候選哪一個？

| 需求                                                | 推薦映像                          |
|-----------------------------------------------------|-----------------------------------|
| Dev/測試環境需要即時 UI 觀察、Prometheus 監控       | `*-management`                    |
| 嚴格的 Production、全自動 IaC/監控已涵蓋，UI 不對外 | 純版號（自行決定是否啟用 plugin） |

> 小技巧：若想在基底映像一次啟用多個插件，可自製 Dockerfile：
>
> ```Dockerfile
> FROM rabbitmq:3.13
> RUN rabbitmq-plugins enable --offline rabbitmq_management rabbitmq_prometheus rabbitmq_top
> ```

如此便能兼顧體積與客製化需求。

[1]: https://hub.docker.com/_/rabbitmq?utm_source=chatgpt.com "rabbitmq - Official Image - Docker Hub"
[2]: https://stackoverflow.com/questions/47290108/how-to-open-rabbitmq-in-browser-using-docker-container?utm_source=chatgpt.com "How to open rabbitmq in browser using docker container?"

### ref

- [RebbitMQ Operator Install](https://www.rabbitmq.com/kubernetes/operator/install-operator)
- [RebbitMQ Install Examples (use operator)](https://www.rabbitmq.com/kubernetes/operator/using-operator#examples)