# k8s-pub
## charmuseum
### Install and run
```
chartmuseum --port=<port> --storage="local" --storage-local-rootdir="<chart-dir>"
```
### Add repo and update
```
helm repo add chartmuseum http://localhost:8080
helm repo update
```
### Download chart directly
```
wget  http://localhost:8081/charts/mysql-1.0.3.tg
```

### Package and upload
```
git clone https://github.com/stakater/chart-mysql.git
cd chart-mysql/mysql
helm package .
ls
curl -v -L --data-binary "@mysql-1.0.3.tgz" http://localhost:8081/api/charts

helm repo update
helm install chartmuseum/mysql
```

## Prometheus
Install using helm chart with as much customization as possibly you know, then you can configure additional settings using crds whose yaml manifests stay in git. The problem with this is when you do helm delete

Components:
kube-state-metrics
exposes k8s objects metrics:
pods, svc, deployment, configmaps, pv, pvc, node status

Accessible via: IP:8080/metrics # Use port-forward to access
Alerts to create:
Check if specific pods/deployements are ready, use AI to create the query


Node_exporter:
IP:9100 # Use port-forward to access


## Grafana:
### Adding new dashboard to values.yaml
```
+dashboards:
+  default:
+    node-exporter:
+      json: |
+        {
```
### Add datasource to values.yaml
```
+datasources:
+  datasources.yaml:
+    apiVersion: 1
+    datasources:
+    - name: Prometheus
+      type: prometheus
+      url: http://prometheus-server.monitoring.svc.cluster.local
+      access: proxy
+      isDefault: true
```
### Add provider to values.yaml
```
+dashboardProviders:
+  dashboardproviders.yaml:
+    apiVersion: 1
+    providers:
+    - name: 'default'
+      orgId: 1
+      folder: ''
+      type: file
+      disableDeletion: false
+      editable: true
+      options:
+        path: /var/lib/grafana/dashboards/default
```
### Get grafana secret
Default: admin/prom-operator
```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 3000
```  
## ingress
```
 ingress:
-  enabled: false
-  # For Kubernetes >= 1.18 you should specify the ingress-controller via the field ingressClassName
-  # See https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/#specifying-the-class-of-an-ingress
-  # ingressClassName: nginx
-  # Values can be templated
-  annotations: {}
-    # kubernetes.io/ingress.class: nginx
-    # kubernetes.io/tls-acme: "true"
-  labels: {}
+  enabled: true
+  ingressClassName: nginx
+  annotations:
+    kubernetes.io/ingress.class: nginx
+    nginx.ingress.kubernetes.io/ssl-redirect: "false"

   hosts:
-    - chart-example.local
+    - grafana.local
```

## Syslog-ng exporter
```
[Unit]
Description=Syslog-ng Prometheus Exporter Service
After=syslog-ng.service

[Service]
EnvironmentFile=-/etc/sysconfig/sngexporter
ExecStart=/usr/bin/python3 /root/sngexporter/sng_exporter.py $SNGEXPORTER_PARAMS
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

## node exporter
```
     sudo useradd --no-create-home --shell /bin/false node_exporter
      wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
      tar xvfz node_exporter-1.8.2linux-amd64.tar.gz
      sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
      sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
      sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
      [Unit]
      Description=Node Exporter
      After=network.target

      [Service]
      User=node_exporter
      ExecStart=/usr/local/bin/node_exporter
      Restart=on-failure

      [Install]
      WantedBy=default.target
      EOF
```
### Reload and start
```
      sudo systemctl daemon-reload
      sudo systemctl start node_exporter
      sudo systemctl enable node_exporter
```

## Adding prometheus scrape configs
Add below to values.yaml
```
    additionalScrapeConfigs:
      - job_name: 'node-exporter-external'
        static_configs:
          - targets: ['192.168.2.194:9100']
        metrics_path: /metrics
        scheme: http
      - job_name: 'syslog-ng'
        static_configs:
          - targets: ['192.168.2.195:9577']
        metrics_path: /metrics
        scheme: http
```

## Add prometheus rule for diskspac
### Provide custom recording or alerting rules to be deployed into the cluster.
```
additionalPrometheusRulesMap:
  external-node-diskspace:
    groups:
    - name: node-exporter-diskspace
      rules:
      - alert: ExternalNodeDiskSpaceWarning
        annotations:
          description: Filesystem on {{ $labels.device }}, mounted on {{ $labels.mountpoint }}, at {{ $labels.instance }} has only {{ printf "%.2f" $value }}% available space left.
          runbook_url: https://runbooks.prometheus-operator.dev/runbooks/node/nodefilesystemalmostoutofspace
          summary: External node disk space is running low (< 10% left)
        expr: |
          (
            node_filesystem_avail_bytes{job="node-exporter-external",fstype!="",mountpoint!=""} / node_filesystem_size_bytes{job="node-exporter-external",fstype!="",mountpoint!=""} * 100 < 10
          and
            node_filesystem_readonly{job="node-exporter-external",fstype!="",mountpoint!=""} == 0
          )
        for: 5m
        labels:
          severity: warning
      - alert: ExternalNodeDiskSpaceCritical
        annotations:
          description: Filesystem on {{ $labels.device }}, mounted on {{ $labels.mountpoint }}, at {{ $labels.instance }} has only {{ printf "%.2f" $value }}% available space left.
          runbook_url: https://runbooks.prometheus-operator.dev/runbooks/node/nodefilesystemalmostoutofspace
          summary: External node disk space is critically low (< 5% left)
        expr: |
          (
            node_filesystem_avail_bytes{job="node-exporter-external",fstype!="",mountpoint!=""} / node_filesystem_size_bytes{job="node-exporter-external",fstype!="",mountpoint!=""} * 100 < 5
          and
            node_filesystem_readonly{job="node-exporter-external",fstype!="",mountpoint!=""} == 0
          )
        for: 5m
        labels:
          severity: critical
```
## Add email alert 
```
// ... existing code ...
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      smtp_from: admin@meelass.com
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_auth_username: 'x@meelass.com'
      smtp_auth_password: 'xxxx'
      smtp_require_tls: true
    
    route:
      group_by: ['alertname', 'job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'email-notifications'
      routes:
        - matchers:
            - alertname =~ "ExternalNodeDiskSpace(Warning|Critical)"
          receiver: 'email-notifications'
          continue: true

    receivers:
      - name: 'email-notifications'
        email_configs:
          - to: 'admin@meelass.com'
            send_resolved: true
            headers:
              subject: '{{ template "email.default.subject" . }}'
            html: '{{ template "email.default.html" . }}'

    templates:
      - '/etc/alertmanager/config/*.tmpl'
// ... existing code ...
```


## Disabling default rules:
Update values.yaml
```
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8sContainerCpuUsageSecondsTotal: true
    k8sContainerMemoryCache: true
    k8sContainerMemoryRss: true
    k8sContainerMemorySwap: true
    k8sContainerResource: true
    k8sContainerMemoryWorkingSetBytes: true
    k8sPodOwner: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubeControllerManager: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeSchedulerAlerting: true
    kubeSchedulerRecording: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true
    windows: true
```
## Kubecontroller manager listen address
```
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
-bind-address=0.0.0.0
systemctl restart  docker
```
## etcd listen address
```
vim /etc/kubernetes/manifests/etcd.yaml
-listen-metrics-urls=http://0.0.0.0:2381
systemctl restart  docker
```
## schedular
```
vim /etc/kubernetes/manifests/kube-scheduler.yam
 --bind-address=0.0.0.0
systemctl restart  docker
```
## kube-proxy
```
kubectl edit cm/kube-proxy -n kube-system
metricsBindAddress: "0.0.0.0:10249"
kubectl rollout restart ds kube-proxy -n kube-system
```

