![Demo](demo.gif)


# Налаштовуємо і запускаємо моніторинг локально за допомогою Flux:
1. Встановлюємо kind:
```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
2. Встановлюємо flux cd:
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```
3. Створюємо кластер за допомогою kind:
```bash
kind create cluster
```
4. Перевіряємо умови для встановлення flux:
```bash
flux check --pre
```
5. Встановлюємо flux до кластеру:
```bash
flux install
```
6. Створюємо репозиторій GitHub для flux (Flux bootstrap for GitHub):
```bash
export GITHUB_TOKEN=[token]

flux bootstrap github --token-auth --owner=[owner name] --repository=[repository name] --branch=main --path=clusters/[cluster name] --personal
```
 7. Створюємо маніфест для розгортання namespace:
 ```yaml
 apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```
  8. Створюємо маніфест для розгортання cert-manager у репозиторії flux:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: cert-manager
  namespace: monitoring
spec:
  interval: 1m0s
  url: https://charts.jetstack.io
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: monitoring
spec:
  chart:
    spec:
      chart: cert-manager
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: cert-manager
        namespace: monitoring
      version: 1.13.x
  interval: 1m0s
  values:
    installCRDs: true
```
9. Створюємо маніфест для розгортання Open Telemetry Operator у репозиторії flux:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: opentelemetry
  namespace: monitoring
spec:
  interval: 1m0s
  url: https://open-telemetry.github.io/opentelemetry-helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: opentelemetry-operator
  namespace: monitoring
spec:
  chart:
    spec:
      chart: opentelemetry-operator
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: opentelemetry
        namespace: monitoring
  interval: 1m0s
  values:
    admissionWebhooks.certManager.autoGenerateCert.enabled: true
    admissionWebhooks.certManager.enabled: false
    manager.featureGates: operator.autoinstrumentation.go.enabled=true
```
10. Створюємо маніфест для розгортання Open Telemetry Collector та Open Telemetry Collector Sidecar у репозиторії flux:
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: opentelemetry
  namespace: monitoring
spec:
  mode: daemonset
  hostNetwork: true
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: "0.0.0.0:4317"
          http:
            endpoint: "0.0.0.0:3030"

    exporters:
      logging:
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
      prometheus:
        endpoint: "0.0.0.0:8889"

    service:
      pipelines:
        logs:
          receivers: [otlp]
          exporters: [loki]
        traces:
          receivers: [otlp]
          exporters: [logging]
        metrics:
          receivers: [otlp]
          exporters: [logging,prometheus]
```
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: opentelemetry-sidecar
  namespace: monitoring
spec:
  mode: sidecar
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: "0.0.0.0:4317"
          http:
            endpoint: "0.0.0.0:3030"
    exporters:
      logging:
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
      prometheus:
        endpoint: "0.0.0.0:8889"
    service:
      pipelines:
        logs:
          receivers: [otlp]
          exporters: [loki]
        traces:
          receivers: [otlp]
          exporters: [logging]
        metrics:
          receivers: [otlp]
          exporters: [logging,prometheus]
```

11. Також розгортаємо Prometheus за допомогою згенерованого маніфесту та ConfigMaps для нього:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: prometheus
  namespace: monitoring
spec:
  interval: 1m0s
  url: https://prometheus-community.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  chart:
    spec:
      chart: prometheus
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: prometheus
        namespace: monitoring
  interval: 1m0s
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    k8s-app: prometheus
data:
  prometheus.yml: |
    global:
      evaluation_interval: 30s
      scrape_interval: 30s
      scrape_timeout: 10s
    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts
    scrape_configs:
    - job_name: otel_collector
      scrape_interval: 5s
      static_configs:
        - targets: ['opentelemetry-collector:8889']
    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - honor_labels: true
      job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - honor_labels: true
      job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    - honor_labels: true
      job_name: prometheus-pushgateway
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - action: keep
        regex: pushgateway
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
    - honor_labels: true
      job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service
    - honor_labels: true
      job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - honor_labels: true
      job_name: kubernetes-pods-slow
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    alerting:
      alertmanagers:
      - kubernetes_sd_configs:
          - role: pod
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          regex: monitoring
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
          regex: prometheus
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
          regex: alertmanager
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex: "9093"
          action: keep
```
12. Далі розгортаємо Fluent-Bit та ConfigMaps для нього:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: fluentbit
  namespace: monitoring
spec:
  interval: 1m0s
  url: https://fluent.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: fluentbit
  namespace: monitoring
spec:
  chart:
    spec:
      chart: fluent-bit
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: fluentbit
        namespace: monitoring
  interval: 1m0s
```

```yaml
---
---
apiVersion: v1
data:
  custom_parsers.conf: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File /fluent-bit/etc/parsers.conf
        Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Exclude_Path      /var/log/containers/*_kube-system_*.log
        multiline.parser  docker, cri
        Refresh_Interval  10
        Ignore_Older      6h
        Docker_Mode       On
        Tag_Regex         (?<pod_name>[^_]+)_(?<namespace_name>[^_]+)_(?<container_name>[^_]+)-(?<docker_id>[a-z0-9]{64})\.log
        Tag               <pod_name>_<namespace_name>_<container_name>-<docker_id>

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Merge_Log_Key log_processed
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            opentelemetry
        Match           *
        Host            opentelemetry-collector
        Port            3030
        metrics_uri     /v1/metrics
        logs_uri        /v1/logs
        Log_response_payload True
        tls             off
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: fluentbit
    meta.helm.sh/release-namespace: monitoring
  labels:
    app.kubernetes.io/instance: fluentbit
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: fluent-bit
    app.kubernetes.io/version: 2.2.1
    helm.sh/chart: fluent-bit-0.42.0
    helm.toolkit.fluxcd.io/name: fluentbit
    helm.toolkit.fluxcd.io/namespace: monitoring
  name: fluentbit-fluent-bit
  namespace: monitoring
```
13. Додаємо Helm-репозиторій для розгортання Grafana та Loki:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: grafana
  namespace: monitoring
spec:
  interval: 1m0s
  url: https://grafana.github.io/helm-charts
```
14. Наступним розгортаємо Loki разом з ConfigMaps для нього:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: loki
  namespace: monitoring
spec:
  chart:
    spec:
      chart: loki
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: monitoring
  interval: 1m0s
  values:
    loki:
      commonConfig:
        replication_factor: 1
      storage:
        type: filesystem
    singleBinary:
      replicas: 1
    memberlist:
      service:
        publishNotReadyAddresses: true
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki
  namespace: monitoring
data:
  config.yaml: |
    auth_enabled: false
    common:
      compactor_address: 'loki'
      path_prefix: /var/loki
      replication_factor: 1
      storage:
        filesystem:
          chunks_directory: /var/loki/chunks
          rules_directory: /var/loki/rules
    frontend:
      scheduler_address: ""
    frontend_worker:
      scheduler_address: ""
    index_gateway:
      mode: ring
    limits_config:
      max_cache_freshness_per_query: 10m
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      split_queries_by_interval: 15m
    memberlist:
      join_members:
      - loki-memberlist
    query_range:
      align_queries_with_step: true
      results_cache:
        cache:
          embedded_cache:
            max_size_mb: 100
            enabled: true
    ruler:
    #   alertmanager_url: http://alertmanager.monitoring.svc:9093
      alertmanager_url: http://localhost:9093
      storage:
        type: local
    runtime_config:
      file: /etc/loki/runtime-config/runtime-config.yaml
    schema_config:
      configs:
      - from: "2022-01-11"
        index:
          period: 24h
          prefix: loki_index_
        object_store: filesystem
        schema: v12
        store: boltdb-shipper
    server:
      grpc_listen_port: 9095
      http_listen_port: 3100
    storage_config:
      hedging:
        at: 250ms
        max_per_second: 20
        up_to: 3
    tracing:
      enabled: false
    analytics:
      reporting_enabled: false
```
15. Далі розгортаємо Grafana:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: grafana
  namespace: monitoring
spec:
  chart:
    spec:
      chart: grafana
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: monitoring
  interval: 1m0s
  values:
    datasources:
      datasources.yaml:
        apiVersion: 1
        datasources:
        - access: proxy
          basicAuth: false
          editable: true
          isDefault: false
          name: Loki
          type: loki
          url: http://loki:3100
          version: 1
          orgId: 1
        - access: proxy
          basicAuth: false
          editable: true
          isDefault: true
          jsonData:
            httpMethod: GET
          name: Prometheus
          orgId: 1
          type: prometheus
          uid: prometheus
          url: http://prometheus-server.monitoring.svc
          version: 1
    env:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_AUTH_DISABLE_LOGIN_FORM: "true"
      GF_FEATURE_TOGGLES_ENABLE: traceqlEditor
      GF_SERVER_PORT: "3000"
```
16. Створюємо маніфест для розгортання боту у репозиторії flux:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: kbot
  namespace: monitoring
spec:
  interval: 1m10s
  ref:
    branch: main
  url: https://github.com/dm-ol/kbot.git
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: kbot
  namespace: monitoring
spec:
  chart:
    spec:
      chart: ./helm
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: GitRepository
        name: kbot
  interval: 1m0s
```
16. Додаємо токен для боту в будь-який зручний спосіб (бажано безпечно).
17. Відкриваємо порт для доступу до Grafana:

```bash
k port-forward service/grafana 3000:80 -n monitoring
```
18. Якщо в процесі розгортання з'явиться помилка розгортання подів `CrashLoopBackOff` з  `Failed to allocate directory watch: Too many open files` в логах:
```bash
sudo sysctl -w fs.inotify.max_user_instances=256
```
19. Якщо все добре, запускаємо Grafana в браузері `localhoct:3000` і налаштовуємо дашбоард.
