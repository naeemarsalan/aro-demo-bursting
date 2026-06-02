# Step-by-step setup (v2 — anaeem LB, virt as metric source)

Hand-run version of `README.md`. Assumes the Custom Metrics Autoscaler
(KEDA) is already installed on anaeem.

All cross-cluster traffic originates from anaeem — in this lab virt cannot
resolve anaeem's apps router, but anaeem can resolve virt's.

```
loadgen (anaeem) ─► nginx LB (anaeem) ─┬─ weight 5 ─► anaeem burst app   (KEDA scaled)
                                       └─ weight 1 ─► virt app via Route
                                                       │  (nginx + exporter sidecar)
                                                       ▼
                                                 virt Prometheus
                                                       ▲
                                                       │ public Route
                                                  KEDA on anaeem
```

## 0. Prereqs

- `oc` (4.14+), cluster admin on both clusters.
- Private network connectivity (172.16/16) between virt and anaeem.
- KEDA already installed on anaeem (see MANUAL.md §4 if not).

## 1. Log in, name the contexts

```bash
oc login -u <user> -p '...' https://api.virt.na-launch.com:6443
oc config rename-context $(oc config current-context) virt

oc login -u <user> -p '...' https://api.anaeem.na-launch.com:6443
oc config rename-context $(oc config current-context) anaeem
```

## 2. Virt: demo namespace + app (nginx + exporter sidecar)

```bash
oc --context virt create ns demo

cat <<'EOF' | oc --context virt apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: burst-app-html, namespace: demo }
data:
  default.conf: |
    server {
      listen 8080;
      location = /healthz { return 200 "ok\n"; }
      location = /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
      }
      location / {
        root /tmp/html;
        try_files /index.html =404;
      }
    }
  index.html.template: |
    <!doctype html><html><head><title>${CLUSTER}</title><meta charset=utf-8>
    <style>body{margin:0;font:18px sans-serif;text-align:center;padding:6em 1em;
    background:${COLOR};color:white;min-height:100vh}
    .box{background:rgba(0,0,0,.25);padding:2em 3em;border-radius:12px;display:inline-block}
    h1{margin:0 0 .5em;font-size:2em}
    code{background:rgba(255,255,255,.15);padding:.2em .4em;border-radius:4px}
    </style></head><body><div class=box>
    <h1>Served from <b>${CLUSTER}</b></h1>
    <p>Pod: <code>${POD_NAME}</code></p>
    </div></body></html>
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: prometheus-example-app, namespace: demo, labels: { app: prometheus-example-app } }
spec:
  replicas: 2
  selector: { matchLabels: { app: prometheus-example-app } }
  template:
    metadata: { labels: { app: prometheus-example-app } }
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27-alpine
          command: ["/bin/sh","-c"]
          args:
            - |
              envsubst '${POD_NAME} ${CLUSTER} ${COLOR}' < /tmpl/index.html.template > /tmp/html/index.html
              exec nginx -g 'daemon off;'
          env:
            - { name: POD_NAME, valueFrom: { fieldRef: { fieldPath: metadata.name } } }
            - { name: CLUSTER, value: virt }
            - { name: COLOR,   value: "#2563eb" }
          ports: [{ name: http, containerPort: 8080 }]
          securityContext: { allowPrivilegeEscalation: false, capabilities: { drop: [ALL] } }
          volumeMounts:
            - { name: tmpl,    mountPath: /tmpl,             readOnly: true }
            - { name: conf,    mountPath: /etc/nginx/conf.d, readOnly: true }
            - { name: tmphtml, mountPath: /tmp/html }
        - name: exporter
          image: docker.io/nginx/nginx-prometheus-exporter:1.3.0
          args: ["--nginx.scrape-uri=http://localhost:8080/stub_status"]
          ports: [{ name: metrics, containerPort: 9113 }]
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            capabilities: { drop: [ALL] }
      volumes:
        - { name: tmpl, configMap: { name: burst-app-html, items: [{ key: index.html.template, path: index.html.template }] } }
        - { name: conf, configMap: { name: burst-app-html, items: [{ key: default.conf,        path: default.conf }] } }
        - { name: tmphtml, emptyDir: {} }
---
apiVersion: v1
kind: Service
metadata: { name: prometheus-example-app, namespace: demo, labels: { app: prometheus-example-app } }
spec:
  selector: { app: prometheus-example-app }
  ports:
    - { name: http,    port: 8080, targetPort: http }
    - { name: metrics, port: 9113, targetPort: metrics }
---
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: prometheus-example-app, namespace: demo }
spec:
  to: { kind: Service, name: prometheus-example-app }
  port: { targetPort: http }
EOF

oc --context virt -n demo rollout status deploy/prometheus-example-app
```

## 3. Virt: Prometheus that scrapes the exporter sidecars

```bash
cat <<'EOF' | oc --context virt apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: prometheus-config, namespace: demo }
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
      - job_name: app
        kubernetes_sd_configs:
          - role: pod
            namespaces: { names: [demo] }
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: prometheus-example-app
            action: keep
          - source_labels: [__meta_kubernetes_pod_container_port_number]
            regex: "9113"
            action: keep
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
---
apiVersion: v1
kind: ServiceAccount
metadata: { name: prometheus, namespace: demo }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: prometheus, namespace: demo }
rules:
  - apiGroups: [""]
    resources: [pods, services, endpoints]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: prometheus, namespace: demo }
roleRef: { kind: Role, name: prometheus, apiGroup: rbac.authorization.k8s.io }
subjects: [{ kind: ServiceAccount, name: prometheus, namespace: demo }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: prometheus, namespace: demo, labels: { app: prometheus } }
spec:
  replicas: 1
  selector: { matchLabels: { app: prometheus } }
  template:
    metadata: { labels: { app: prometheus } }
    spec:
      serviceAccountName: prometheus
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
        - name: prometheus
          image: quay.io/prometheus/prometheus:v2.53.1
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --storage.tsdb.path=/prometheus
            - --storage.tsdb.retention.time=1h
            - --web.listen-address=:9090
          ports: [{ name: web, containerPort: 9090 }]
          securityContext:
            allowPrivilegeEscalation: false
            capabilities: { drop: [ALL] }
          volumeMounts:
            - { name: config, mountPath: /etc/prometheus, readOnly: true }
            - { name: data,   mountPath: /prometheus }
      volumes:
        - { name: config, configMap: { name: prometheus-config } }
        - { name: data,   emptyDir: {} }
---
apiVersion: v1
kind: Service
metadata: { name: prometheus, namespace: demo, labels: { app: prometheus } }
spec:
  selector: { app: prometheus }
  ports: [{ name: web, port: 9090, targetPort: web }]
---
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: prometheus, namespace: demo }
spec:
  to: { kind: Service, name: prometheus }
  port: { targetPort: web }
  tls: { termination: edge, insecureEdgeTerminationPolicy: Redirect }
EOF

oc --context virt -n demo rollout status deploy/prometheus

PROM_URL="https://$(oc --context virt -n demo get route prometheus -o jsonpath='{.spec.host}')"
echo "PROM_URL=$PROM_URL"
# Confirm both pods scraped
curl -sk "$PROM_URL/api/v1/targets" | python3 -c "import sys,json; [print(t['scrapeUrl'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"
```

## 4. Anaeem: demo namespace + burst app (replicas: 0)

```bash
oc --context anaeem create ns demo

cat <<'EOF' | oc --context anaeem apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: burst-app-html, namespace: demo }
data:
  default.conf: |
    server {
      listen 8080;
      location = /healthz { return 200 "ok\n"; }
      location / {
        root /tmp/html;
        try_files /index.html =404;
      }
    }
  index.html.template: |
    <!doctype html><html><head><title>${CLUSTER}</title><meta charset=utf-8>
    <style>body{margin:0;font:18px sans-serif;text-align:center;padding:6em 1em;
    background:${COLOR};color:white;min-height:100vh}
    .box{background:rgba(0,0,0,.25);padding:2em 3em;border-radius:12px;display:inline-block}
    h1{margin:0 0 .5em;font-size:2em}
    code{background:rgba(255,255,255,.15);padding:.2em .4em;border-radius:4px}
    </style></head><body><div class=box>
    <h1>Served from <b>${CLUSTER}</b></h1>
    <p>Pod: <code>${POD_NAME}</code></p>
    </div></body></html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-example-app
  namespace: demo
  labels: { app: prometheus-example-app, role: burst }
spec:
  replicas: 0
  selector: { matchLabels: { app: prometheus-example-app } }
  template:
    metadata: { labels: { app: prometheus-example-app, role: burst } }
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27-alpine
          command: ["/bin/sh","-c"]
          args:
            - |
              envsubst '${POD_NAME} ${CLUSTER} ${COLOR}' < /tmpl/index.html.template > /tmp/html/index.html
              exec nginx -g 'daemon off;'
          env:
            - { name: POD_NAME, valueFrom: { fieldRef: { fieldPath: metadata.name } } }
            - { name: CLUSTER, value: anaeem }
            - { name: COLOR,   value: "#16a34a" }
          ports: [{ name: http, containerPort: 8080 }]
          securityContext: { allowPrivilegeEscalation: false, capabilities: { drop: [ALL] } }
          volumeMounts:
            - { name: tmpl,    mountPath: /tmpl,             readOnly: true }
            - { name: conf,    mountPath: /etc/nginx/conf.d, readOnly: true }
            - { name: tmphtml, mountPath: /tmp/html }
      volumes:
        - { name: tmpl, configMap: { name: burst-app-html, items: [{ key: index.html.template, path: index.html.template }] } }
        - { name: conf, configMap: { name: burst-app-html, items: [{ key: default.conf,        path: default.conf }] } }
        - { name: tmphtml, emptyDir: {} }
---
apiVersion: v1
kind: Service
metadata: { name: prometheus-example-app, namespace: demo }
spec:
  selector: { app: prometheus-example-app }
  ports: [{ name: http, port: 8080, targetPort: http }]
EOF
```

No Route on the burst app — it's only reachable through the local LB.

## 5. Anaeem: weighted nginx LB (5 local : 1 virt)

```bash
cat <<'EOF' | oc --context anaeem apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: lb-config, namespace: demo }
data:
  default.conf: |
    upstream backend {
      server prometheus-example-app.demo.svc.cluster.local:8080 weight=5 max_fails=2 fail_timeout=10s;
      server prometheus-example-app-demo.apps.virt.na-launch.com:80 weight=1 max_fails=2 fail_timeout=10s;
    }
    server {
      listen 8080;
      proxy_set_header Host prometheus-example-app-demo.apps.virt.na-launch.com;
      proxy_next_upstream error timeout http_502 http_503 http_504;
      proxy_connect_timeout 2s;
      proxy_read_timeout 5s;
      location / { proxy_pass http://backend; }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: lb, namespace: demo, labels: { app: lb } }
spec:
  replicas: 1
  selector: { matchLabels: { app: lb } }
  template:
    metadata: { labels: { app: lb } }
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27-alpine
          ports: [{ name: http, containerPort: 8080 }]
          securityContext: { allowPrivilegeEscalation: false, capabilities: { drop: [ALL] } }
          volumeMounts:
            - { name: conf, mountPath: /etc/nginx/conf.d, readOnly: true }
      volumes:
        - { name: conf, configMap: { name: lb-config } }
---
apiVersion: v1
kind: Service
metadata: { name: lb, namespace: demo }
spec:
  selector: { app: lb }
  ports: [{ name: http, port: 80, targetPort: http }]
---
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: lb, namespace: demo }
spec:
  to: { kind: Service, name: lb }
  port: { targetPort: http }
  tls: { termination: edge, insecureEdgeTerminationPolicy: Redirect }
EOF

oc --context anaeem -n demo rollout status deploy/lb
LB_URL="https://$(oc --context anaeem -n demo get route lb -o jsonpath='{.spec.host}')"
echo "LB_URL=$LB_URL"
```

## 6. Anaeem: ScaledObject pointed at virt's Prometheus

```bash
cat <<EOF | oc --context anaeem apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: demo-burst, namespace: demo }
spec:
  scaleTargetRef: { name: prometheus-example-app }
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 0
  maxReplicaCount: 5
  triggers:
    - type: prometheus
      metadata:
        serverAddress: ${PROM_URL}
        query: sum(rate(nginx_http_requests_total[1m]))
        threshold: "5"
        activationThreshold: "2"
        unsafeSsl: "true"
EOF

oc --context anaeem -n demo get scaledobject demo-burst
```

## 7. Drive load on anaeem

```bash
cat <<'EOF' | oc --context anaeem apply -f -
apiVersion: v1
kind: Pod
metadata: { name: loadgen-pod, namespace: demo, labels: { app: loadgen } }
spec:
  restartPolicy: Never
  containers:
    - name: hey
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["/bin/sh","-c"]
      args:
        - |
          end=$(( $(date +%s) + 900 ))
          while [ "$(date +%s)" -lt "$end" ]; do
            for i in $(seq 1 30); do
              curl -s -o /dev/null http://lb.demo.svc:80/ &
            done
            wait
          done
EOF

oc --context anaeem -n demo wait --for=condition=Ready pod/loadgen-pod --timeout=60s
```

## 8. Watch the burst

```bash
watch -n 2 'oc --context anaeem -n demo get scaledobject demo-burst; \
  oc --context anaeem -n demo get hpa keda-hpa-demo-burst; \
  oc --context anaeem -n demo get pods -l app=prometheus-example-app'

for i in $(seq 1 60); do curl -sk "$LB_URL/" | grep -oE 'from <b>[a-z]+</b>'; done | sort | uniq -c
```

Timeline:
1. Initially anaeem has 0 burst pods; the LB falls back to virt (all blue).
2. virt's exporter rate climbs; KEDA scales anaeem 0→1→…→5 in ~30–90s.
3. After the 10s `fail_timeout` expires, the LB starts using the anaeem
   upstream again — ~5/6 of samples become green (anaeem), 1/6 blue (virt).

## 9. Stop load + teardown

```bash
oc --context anaeem -n demo delete pod loadgen-pod --grace-period=0 --force
# ~60s cooldown later, anaeem scales back to 0; LB returns to all-virt.

oc --context anaeem -n demo delete scaledobject demo-burst
oc --context anaeem delete ns demo
oc --context virt   delete ns demo
```

## Troubleshooting

- **All requests stay blue (virt) even after anaeem pods are Ready.**
  nginx caches upstream "down" state for `fail_timeout` (10s). Wait it out,
  or `oc -n demo rollout restart deploy/lb` on anaeem.

- **All requests to anaeem upstream get 503 even with burst pods Running.**
  Check Host header — the LB must send `Host: <virt route hostname>` so
  virt's OpenShift router matches. anaeem's local Service ignores Host, so
  this fixed value works for both upstreams.

- **`Active=False` despite loadgen running.**
  `curl -sk "$PROM_URL/api/v1/query?query=sum(rate(nginx_http_requests_total[1m]))"` —
  empty means virt's exporter sidecar isn't being scraped. Check
  `oc -n demo get pods` and the Prometheus targets:
  `curl -sk "$PROM_URL/api/v1/targets"`.

- **Browser hits `http://lb-demo...` and gets 503.**
  Browsers auto-upgrade to HTTPS; the LB Route uses edge-TLS with
  `insecureEdgeTerminationPolicy: Redirect` (step 5 includes this).
