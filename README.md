# This repository contains the solutions for the SRE pretest assignment from Dynasafe

## 架構說明

![Dynasafe-pretest-arch](https://github.com/user-attachments/assets/11457b7c-9f8e-44d7-86a0-7af946daeada)

1. 在Kind cluster中創建4個node, 1個control-plan, 3個worker node
2. 其中一個worker node當作infra node, 安裝prometheus, metallb, speaker
   ```
   helm install metallb metallb/metallb --namespace metallb-system --create-namespace --set speaker.nodeSelector.node-type=infra
   ```
   ```
   helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace --set server.nodeSelector."node-type"=infra  --set kubeStateMetrics.nodeSelector."node-type"=infra
   ```
   ```
   helm install node-exporter prometheus-community/prometheus-node-exporter --namespace monitoring --set nodeSelector."node-type"=infra
   ```
  
4. 創建一個nginx deployment並且創建一個svc給nginx使用，因為有metallb, 因此type可使用LoadBalancer。
   不過由於本機是mac, docker似乎不會向主機公開docker的網路([來源](https://stackoverflow.com/questions/75512091/cannot-access-load-balancer-external-ip-address-assigned-by-metallb-installed-on))
5. 目前架構較簡易，所以Prometheus的9090port先用forwarding的方式，mapping到localhost:9090 port上
6. Grafana run 在Docker container內，因此 Data source無法直接訪問`localhost:9090`, 需要訪問宿主機`host.docker.internal`，才能拿到上一步forwarding的prometheus endpoint
7. 同時Grafana container mapping port3000到`localhost:3000`，因此在本機3000port就可以訪問Grafana Dashboard

---

## 相關效能數據
1. 呈現node的效能監控數據
  <img width="1305" alt="image" src="https://github.com/user-attachments/assets/fff2b484-447d-44df-9b0d-65f68e42f63c" />


```
sum(irate(node_cpu_seconds_total{instance="$node",job="$job", mode="system"}[$__rate_interval])) / scalar(count(count(node_cpu_seconds_total{instance="$node",job="$job"}) by (cpu)))
```

2. 呈現 cluster的效能監控數據
  <img width="1480" alt="image" src="https://github.com/user-attachments/assets/17ed93ca-d278-4897-b504-0f7d79a34849" />

```
rate(apiserver_request_total{resource!="", group!="", verb!~"CONNECT|WATCH|WATCHLIST"}[5m])
```

3. 呈現 USE(Utilization, Saturation, Errors)角度的效能監控數據
  <img width="1492" alt="image" src="https://github.com/user-attachments/assets/ab6d2a01-25cb-440a-af35-42611a675954" />

```
instance:node_load1_per_cpu:ratio{instance="$instance"}
```

4. 呈現 etcd的效能監控數據
  <img width="1480" alt="image" src="https://github.com/user-attachments/assets/9c164c70-73f4-4fc0-b0bf-f177049d6672" />

```
sum(rate(grpc_server_started_total{grpc_type="unary"}[5m]))
```

5. 呈現 Prometheus效能監控數據
   <img width="1481" alt="image" src="https://github.com/user-attachments/assets/0767f035-68ed-4ac5-8316-cba62f86af96" />
```
prometheus_tsdb_head_series{job="kube-prometheus-stack-prometheus",instance="$Prometheus:9090"}
```

6. CPU Throttling監控數據
<img width="1166" alt="image" src="https://github.com/user-attachments/assets/dd95ba3d-509a-47c1-a8b5-c01400edec98" />

```
sum(increase(container_cpu_cfs_throttled_periods_total{container!="",}[10m])) by(cluster, container,pod, namespace)/
sum(increase(container_cpu_cfs_periods_total{}[10m])) by (cluster, container, pod, namespace) * 100
```
promql 參考來源：Grafana官方自己有一篇關於CPU throttling的[文章](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/optimize-resource-usage/cpu-throttling/)

提示我們可以使用這兩個metrics，來算出某個container被限制的時間比率，由此來設計告警。

---

## 相關配置檔放置於目錄manifest中

   
