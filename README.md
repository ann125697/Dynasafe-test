# Kind Kubernetes Cluster Setup and Monitoring
[TOC]

## 1. Kind Cluster Setup

 使用 Kind 架設一個 3 個節點的 control-plane 和 2 個節點的 worker。

### 設定 Kind Cluster YAML

```yaml
# kind-cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
```
### 建立 Kind Cluster
```bash
kind create cluster --config kind-cluster-config.yaml
```
## 2. 安裝 Prometheus、Node Exporter 和 Kube-State-Metrics
安裝 Prometheus, node exporter, kube-state-metrics 在kind的叢集裡，Prometheus 收集node exporter, kube-state-metrics的效能數據。
### 使用 Helm 安裝 Prometheus
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus --create-namespace --namespace monitor
```
```bash
kubectl get all -n monitor
```

### Ingress NGINX
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

#### 建立 prometheus Ingress
```yam=
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitor
spec:
  rules:
  - host: prometheus.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prometheus-server
            port:
              number: 80
```

## 3. 安裝 Grafana 和配置儀表板
### 使用 Docker 啟動 Grafana
> 使用與 kind 同一個 docker network

get cluster info
```bash
kubectl get nodes -o wide          
NAME                  STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION    CONTAINER-RUNTIME
kind-control-plane    Ready    control-plane   11m   v1.31.0   172.18.0.8    <none>        Debian GNU/Linux 12 (bookworm)   6.10.0-linuxkit   containerd://1.7.18
kind-control-plane2   Ready    control-plane   11m   v1.31.0   172.18.0.5    <none>        Debian GNU/Linux 12 (bookworm)   6.10.0-linuxkit   containerd://1.7.18
kind-control-plane3   Ready    control-plane   11m   v1.31.0   172.18.0.7    <none>        Debian GNU/Linux 12 (bookworm)   6.10.0-linuxkit   containerd://1.7.18
kind-worker           Ready    <none>          11m   v1.31.0   172.18.0.6    <none>        Debian GNU/Linux 12 (bookworm)   6.10.0-linuxkit   containerd://1.7.18
kind-worker2          Ready    <none>          11m   v1.31.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.10.0-linuxkit   containerd://1.7.18
```

編輯 docker-compose
```bash
mkdir grafana;cd grafana
nano docker-compose.yaml
```
```yaml=
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    networks:
      - kind
    extra_hosts:
      - "prometheus.com:172.18.0.8" #kind-control-plane ip 
    volumes:
      - $PWD/data:/var/lib/grafana
    restart: unless-stopped

networks:
  kind:
    external: true
```
```bash
docker compose up -d
```

* 打開 Grafana Web (http://localhost:3000)
* 登入後，點擊 Configuration -> Data Sources -> Add data source
* 選擇 Prometheus，並填入 Prometheus server URL http://prometheus.com

### 3.1 效能監控儀表板(1): 呈現node的效能監控數據
* 點擊 Dashboards -> New -> New dashboard -> Import dashboard
* 輸入官方提供 dashboard ID (8171) -> Load
![image](https://hackmd.io/_uploads/BJzXNf7n0.png)
![image](https://hackmd.io/_uploads/ByLHVG72C.png)

### 3.2 效能監控儀表板(2): 呈現kind 叢集的效能監控數據
* 點擊 Dashboards -> New -> New dashboard -> Import dashboard
* 輸入官方提供 dashboard ID (3119) -> Load
![image](https://hackmd.io/_uploads/rk_lSfQ20.png)
![image](https://hackmd.io/_uploads/HJtzHf7hR.png)
* 修改無資料呈現的 PromQL

### 3.3 請說明以上的2個效能監控儀表板的每個panel內容。

#### 3.3.1 Kubernetes Nodes

##### Idle CPU(閒置 CPU)
![image](https://hackmd.io/_uploads/r1ZskX7hR.png)
>查詢 Node 最近 5 分鐘內 CPU 的使用率。
```promQL!
100 - (avg by (cpu) (irate(node_cpu_seconds_total{mode="idle", instance="$server"}[5m])) * 100)
```

---

#### System Load (系統負載)
![image](https://hackmd.io/_uploads/H1kxxXm20.png)
>查詢 Node 1分鐘/5分鐘/15分鐘 平均系統負載
>數值越高代表系統越繁忙，運行中及等待運行的 process 數量越多
```promQL!
node_load1{instance="$server"}
```
```promQL!
node_load5{instance="$server"}
```
```promQL!
node_load15{instance="$server"}
```

---


#### Memory Usage (內存使用)
![image](https://hackmd.io/_uploads/HkDAFzQhA.png)
- 查詢計算實際內存使用量（以bytes為單位）
- 減掉空閒內存(node_memory_MemFree_bytes)、緩存內存(node_memory_Buffers_bytes)和緩衝區的內存(node_memory_Cached_bytes)來得到正在使用的內存。
```promQL!
node_memory_MemTotal_bytes{instance="$server"} - node_memory_MemFree_bytes{instance="$server"} - node_memory_Buffers_bytes{instance="$server"} - node_memory_Cached_bytes{instance="$server"}
```
```promQL!
node_memory_Buffers_bytes{instance="$server"}
```
```promQL!
node_memory_Cached_bytes{instance="$server"}
```
```promQL!
node_memory_MemFree_bytes{instance="$server"}
```

---

#### Memory Usage (memory使用百分比)
![image](https://hackmd.io/_uploads/SJjxWmXhA.png)
- 查詢 Node memory 使用百分比。
- 正在使用的memory除以總memory，乘以100得到使用百分比。
```promQL!
((node_memory_MemTotal_bytes{instance="$server"} - node_memory_MemFree_bytes{instance="$server"}  - node_memory_Buffers_bytes{instance="$server"} - node_memory_Cached_bytes{instance="$server"}) / node_memory_MemTotal_bytes{instance="$server"}) * 100
```

---

#### Disk I/O (磁盤 I/O)
![image](https://hackmd.io/_uploads/HJHQX77hC.png)
- Node 在最近2分鐘內每秒讀取、寫入的磁盤bytes數量及硬碟IO時間。
- 用 rate() 函數計算時間內的平均讀取、寫入速率。
```promQL!
sum by (instance) (rate(node_disk_read_bytes_total{instance="$server"}[2m]))
```
```promQL!
sum by (instance) (rate(node_disk_written_bytes_total{instance="$server"}[2m]))
```
```promQL!
sum by (instance) (rate(node_disk_io_time_seconds_total{instance="$server"}[2m]))
```

---

#### Disk Space Usage (磁盤空間使用)
![image](https://hackmd.io/_uploads/HJGbNmX3A.png)
- Node硬碟空間的使用百分比。
```promQL!
(sum(node_filesystem_size_bytes{device!="rootfs",instance="$server"}) - sum(node_filesystem_free_bytes{device!="rootfs",instance="$server"})) / sum(node_filesystem_size_bytes{device!="rootfs",instance="$server"})
```

---

#### Network Received (網絡接收)
![image](https://hackmd.io/_uploads/SkPUVQm30.png)
- Node在最近5分鐘內每秒接收到的網絡流量（以bytes為單位）。
- 排除lo,因為通常不是外部網絡流量。
```promQL!
rate(node_network_receive_bytes_total{instance="$server",device!~"lo"}[5m])
```
---

#### Network Transmitted (網絡發送)
![image](https://hackmd.io/_uploads/Hymh8mX20.png)
- 查詢Node在最近5分鐘內每秒發送的網絡流量（以bytes為單位）。
- 也排除 lo
```promQL!
rate(node_network_transmit_bytes_total{instance="$server",device!~"lo"}[5m])
```

---

#### 3.3.2 Kubernetes cluster monitoring

##### Network I/O pressure
![image](https://hackmd.io/_uploads/rkj3WNmnA.png)
- 計算網路 I/O 壓力，透過容器的接收和傳輸網路流量的差值來衡量
- sum(rate(container_network_receive_bytes_total{kubernetes_io_hostname=~"^$Node$"}[$interval]))：Node上的容器接收到的網路流量總和，按速率（每秒bytes數）計算。
- sum(rate(container_network_transmit_bytes_total{kubernetes_io_hostname=~"^$Node$"}[$interval]))：node上的container 傳輸的網路流量總和，按速率計算。
- 最後，兩者相減，得出網路 I/O 壓力的值。

```promQL!
sum (rate (container_network_receive_bytes_total{kubernetes_io_hostname=~"^$Node$"}[$interval]))
```
```promQL!
sum (rate (container_network_transmit_bytes_total{kubernetes_io_hostname=~"^$Node$"}[$interval]))
```

--- 


#### Cluster memory usage
![image](https://hackmd.io/_uploads/B1v3zVmhC.png)
- 計算叢集的記憶體使用率。
- sum(container_memory_working_set_bytes{id="/",kubernetes_io_hostname=~"^$Node$"})：Node上的容器實際使用的記憶體（不包括已釋放但尚未回收的記憶體）。
- sum(machine_memory_bytes{kubernetes_io_hostname=~"^$Node$"}) * 100：該節點的總記憶體容量。兩者相除並乘以100，得出記憶體使用率的百分比。
```promQL!
sum (container_memory_working_set_bytes{id="/",kubernetes_io_hostname=~"^$Node$"}) / sum (machine_memory_bytes{kubernetes_io_hostname=~"^$Node$"}) * 100
```

##### Used
- 叢集中已使用的記憶體
```promQL!
sum (container_memory_working_set_bytes{id="/",kubernetes_io_hostname=~"^$Node$"})
```
##### Total
-  叢集中可用的總記憶體

```promQL!
sum (machine_memory_bytes{kubernetes_io_hostname=~"^$Node$"})
```

--- 

#### Cluster CPU usage ($interval avg)
![image](https://hackmd.io/_uploads/HyQGV4Q2C.png)
- 叢集的 CPU 使用率，按時間間隔（$interval）計算平均值。
- sum(rate(container_cpu_usage_seconds_total{id="/",kubernetes_io_hostname=~"^$Node$"}[$interval]))：Node上的容器使用的 CPU 時間（以秒為單位），按速率（每秒使用的 CPU 時間）計算。
- sum(machine_cpu_cores{kubernetes_io_hostname=~"^$Node$"}) * 100：Node上的總 CPU 核心數，並將兩者相除，得出 CPU 使用率的百分比。
```promQL!
sum (rate (container_cpu_usage_seconds_total{id="/",kubernetes_io_hostname=~"^$Node$"}[$interval])) / sum (machine_cpu_cores{kubernetes_io_hostname=~"^$Node$"}) * 100
```

##### Used
- 叢集中已使用的 CPU
```promQL!
sum (rate (container_cpu_usage_seconds_total{id="/",kubernetes_io_hostname=~"^$Node$"}[$interval]))
```
##### Total
- 叢集中可用的總 CPU 核心數
```promQL!
sum (machine_cpu_cores{kubernetes_io_hostname=~"^$Node$"})
```
---
#### Cluster filesystem usage
![image](https://hackmd.io/_uploads/Bk6mEEX20.png)
- 叢集的檔案系統使用率。

```promQL!
sum (container_fs_usage_bytes{device=~"^/dev/vda.$",id="/",kubernetes_io_hostname=~"^$Node$"}) / sum (container_fs_limit_bytes{device=~"^/dev/vda.$",id="/",kubernetes_io_hostname=~"^$Node$"}) * 100
```

##### Used
- 叢集中已使用的儲存空間
```promQL!
sum (container_fs_usage_bytes{device=~"^/dev/vda.$",id="/",kubernetes_io_hostname=~"^$Node$"})
```
##### Total
- 叢集中總儲存空間
```promQL!
sum (container_fs_limit_bytes{device=~"^/dev/vda.$",id="/",kubernetes_io_hostname=~"^$Node$"})
```
---
#### Pods CPU usage ($interval avg)
![image](https://hackmd.io/_uploads/HJylIv7hR.png)

- Pods 的 CPU 使用情況，按時間間隔（$interval）計算平均值
```promQL!
sum (rate (container_cpu_usage_seconds_total{image!="",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```

---

#### System services CPU usage ($interval avg)
![image](https://hackmd.io/_uploads/B1t1OD730.png)

- Pods 的 CPU 使用情況，按時間間隔（$interval）計算平均值
```promQL!
sum (rate (container_cpu_usage_seconds_total{kubernetes_io_hostname=~"kind-control-plane.*",pod!="",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```

---

#### Containers CPU usage ($interval avg)
![image](https://hackmd.io/_uploads/Bk2aSEQ3A.png)
- 容器的 CPU 使用情況，按時間間隔（$interval）計算平均值
```promQL!
sum (rate (container_cpu_usage_seconds_total{image!="",name=~"^k8s_.*",container_name!="POD",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (container_name, pod_name)
```

---

#### All processes CPU usage ($interval avg)
![image](https://hackmd.io/_uploads/rJ6kIE72C.png)
- 所有 processes 的 CPU 使用情況，按時間間隔（$interval）計算平均值。
```promQL!
sum (rate (container_cpu_usage_seconds_total{id!="/",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```

---

#### Pods memory usage
![image](https://hackmd.io/_uploads/BykrUP7n0.png)

- 計算 Pods 的記憶體使用情況
```promQL!
sum (container_memory_working_set_bytes{image!="",kubernetes_io_hostname=~"^$Node$"}) by (id)
```

---

#### Pods
![image](https://hackmd.io/_uploads/S1IlFPmhC.png)

- Container 在指定Node上的實際memory使用量
```promQL!
sum (container_memory_working_set_bytes{kubernetes_io_hostname=~"kind-control-plane.*",id!="",kubernetes_io_hostname=~"^$Node$"}) by (id)
```

---


#### Containers memory usage
![image](https://hackmd.io/_uploads/S1JSLN7nC.png)
- Container 的記憶體使用情況
```promQL!
sum (container_memory_working_set_bytes{image!="",name=~"^k8s_.*",container_name!="POD",kubernetes_io_hostname=~"^$Node$"}) by (container_name, pod_name)
```

```promQL!
sum (container_memory_working_set_bytes{image!="",name!~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}) by (kubernetes_io_hostname, name, image)
```

```promQL!
sum (container_memory_working_set_bytes{rkt_container_name!="",kubernetes_io_hostname=~"^$Node$"}) by (kubernetes_io_hostname, rkt_container_name)
```

---

#### All processes memory usage
![image](https://hackmd.io/_uploads/rkdOINQhR.png)
- 所有 processes 的記憶體使用情況
```promQL!
sum (container_memory_working_set_bytes{id!="/",kubernetes_io_hostname=~"^$Node$"}) by (id)
```

---

#### Pods network I/O ($interval avg)
![image](https://hackmd.io/_uploads/HJGlvPX3A.png)

-  Pods 的網路 I/O 情況，按時間間隔（$interval）計算平均值。
```promQL!
sum (rate (container_network_receive_bytes_total{image!="",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```

```promQL!
sum (rate (container_network_transmit_bytes_total{image!="",name=~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```

---
 

#### Containers network I/O ($interval avg)
![image](https://hackmd.io/_uploads/H1XJOVQh0.png)
- 容器的網路 I/O 情況，按時間間隔（$interval）計算平均值。
- 包括容器名稱、Pod 名稱的接收和傳輸的流量總和差。
B
```promQL!
sum (rate (container_network_receive_bytes_total{image!="",name=~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (container_name, id)
```
D
```promQL!
sum (rate (container_network_transmit_bytes_total{image!="",name=~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (container_name, id)
```
A
```promQL!
sum (rate (container_network_receive_bytes_total{image!="",name!~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (kubernetes_io_hostname, name, image)
```
C
```promQL!
sum (rate (container_network_transmit_bytes_total{image!="",name!~"^k8s_.*",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (kubernetes_io_hostname, name, image)
```
E
```promQL!
sum (rate (container_network_transmit_bytes_total{rkt_container_name!="",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (kubernetes_io_hostname, rkt_container_name)
```
F
```promQL!
sum (rate (container_network_transmit_bytes_total{rkt_container_name!="",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (kubernetes_io_hostname, rkt_container_name)
```

---

#### All processes network I/O ($interval avg)
![image](https://hackmd.io/_uploads/B14TDEX2R.png)
- 所有 processes 的網路 I/O 情況，按時間間隔（$interval）計算平均值。
- 包括進程 ID 的接收和傳輸的流量總和差。
A
```promQL!
sum (rate (container_network_receive_bytes_total{id!="/",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```
B
```promQL!
sum (rate (container_network_transmit_bytes_total{id!="/",kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```


---



### 3.4 請說明要如何透過建立的監控儀表板觀察CPU Throttling現象。

新增儀表板
```promQL!
sum(rate(container_cpu_cfs_throttled_seconds_total{id!="/", kubernetes_io_hostname=~"^$Node$"}[$interval])) by (id)
```
![image](https://hackmd.io/_uploads/HJkCWvmnC.png)


可以再到worker確認這個container是屬於哪個pod
![image](https://hackmd.io/_uploads/rypeMDm3C.png)


## 4. 請部署一個容器應用程式在kind叢集，建立一個hpa物件以cpu 使用率到達50%為條件，最多擴充到10個pod。

### 安裝 Metrics-server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
```bash
kubectl edit deployment metrics-server -n kube-system

:::找到以下部分，並新增--kubelet-insecure-tls
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # 新增
```
**Do not verify the CA of serving certificates presented by Kubelets. For testing purposes only.**
[Kubernetes Metrics Server](https://kubernetes-sigs.github.io/metrics-server/)

### HorizontalPodAutoscaler Walkthrough
#### Run and expose php-apache server
```bash
nano php-apache.yaml
```
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```
```bash
kubectl apply -f php-apache.yaml
```

#### Create the HorizontalPodAutoscaler
```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
```bash
# You can use "hpa" or "horizontalpodautoscaler"; either name works OK.
kubectl get hpa
```
```
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   cpu: 0%/50%   1         10        1          20h
```



#### Increase the load
```bash!
# Run this in a separate terminal
# so that the load generation continues and you can carry on with the rest of the steps
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Then run:
```bash
# type Ctrl+C to end the watch when you're ready
kubectl get hpa php-apache --watch
```
```
NAME         REFERENCE               TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   cpu: <unknown>/50%   1         10        1          11m
php-apache   Deployment/php-apache   cpu: 0%/50%          1         10        1          11m
php-apache   Deployment/php-apache   cpu: 245%/50%        1         10        1          11m
php-apache   Deployment/php-apache   cpu: 246%/50%        1         10        4          12m
php-apache   Deployment/php-apache   cpu: 112%/50%        1         10        5          12m
php-apache   Deployment/php-apache   cpu: 124%/50%        1         10        5          12m
php-apache   Deployment/php-apache   cpu: 96%/50%         1         10        5          12m
php-apache   Deployment/php-apache   cpu: 56%/50%         1         10        6          13m
php-apache   Deployment/php-apache   cpu: 56%/50%         1         10        6          13m
php-apache   Deployment/php-apache   cpu: 52%/50%         1         10        7          13m
php-apache   Deployment/php-apache   cpu: 46%/50%         1         10        7          13m
php-apache   Deployment/php-apache   cpu: 42%/50%         1         10        7          14m
php-apache   Deployment/php-apache   cpu: 46%/50%         1         10        7          14m
php-apache   Deployment/php-apache   cpu: 42%/50%         1         10        7          14m
php-apache   Deployment/php-apache   cpu: 47%/50%         1         10        7   
```

