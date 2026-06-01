# Step-by-step setup (v2 — weighted LB across virt + anaeem)

Same idea as `MANUAL.md`, with two changes:

- Skips the KEDA operator install — assumes the Custom Metrics Autoscaler is
  already installed on anaeem (was done once in v1).
- Adds an nginx load balancer on virt with **weighted upstreams**, sending
  most traffic to the burst cluster when its pods are up. Each app pod now
  serves a webpage showing its cluster name and pod name.

```
loadgen ─► nginx LB (weights 1:5) ─┬─► virt local Service  (blue page)
                                   └─► anaeem app Route    (green page)
                                          ▲
                                          │ KEDA scales 0→5 based on
                                          │ nginx_http_requests_total
                                          │ from LB exporter sidecar
```

## 0. Prereqs

- `oc` (4.14+), cluster admin on both clusters.
- Private network connectivity between virt and anaeem on 172.16.1.0/24.
- KEDA already installed on anaeem (see MANUAL.md §4 if it isn't).

## 1. Log in, name the contexts

```bash
oc login -u <user> -p '...' https://api.virt.na-launch.com:6443
oc config rename-context $(oc config current-context) virt

oc login -u <user> -p '...' https://api.anaeem.na-launch.com:6443
oc config rename-context $(oc config current-context) anaeem
```

## 2. Virt: demo namespace + app (nginx + HTML)

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
  labels: { app: prometheus-example-app }
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
  ports: [{ name: http, port: 8080, targetPort: http }]
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

## 3. Anaeem: demo namespace + burst app (replicas: 0)

Same pod template, just `CLUSTER=anaeem`, green color, `replicas: 0`.

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
---
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: prometheus-example-app, namespace: demo }
spec:
  to: { kind: Service, name: prometheus-example-app }
  port: { targetPort: http }
EOF
```

The Route exists immediately but returns **503** until KEDA scales the
deployment up — that's what the nginx LB will see when bursting is idle.

## 4. Virt: weighted nginx LB + exporter sidecar

```bash
cat <<'EOF' | oc --context virt apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: lb-config, namespace: demo }
data:
  default.conf: |
    upstream backend {
      # On-prem local Service — low weight, used as fallback
      server prometheus-example-app.demo.svc.cluster.local:8080 weight=1 max_fails=2 fail_timeout=10s;
      # Burst cluster via apps router — high weight, prefer when active
      server prometheus-example-app-demo.apps.anaeem.na-launch.com:80 weight=5 max_fails=2 fail_timeout=10s;
    }
    server {
      listen 8080;
      # OpenShift router on anaeem matches on Host; virt's Service ignores
      # Host. A single fixed value works for both upstreams.
      proxy_set_header Host prometheus-example-app-demo.apps.anaeem.na-launch.com;
      proxy_next_upstream error timeout http_502 http_503 http_504;
      proxy_connect_timeout 2s;
      proxy_read_timeout 5s;
      location = /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
      }
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
        - name: exporter
          image: docker.io/nginx/nginx-prometheus-exporter:1.3.0
          args: ["--nginx.scrape-uri=http://localhost:8080/stub_status"]
          ports: [{ name: metrics, containerPort: 9113 }]
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            capabilities: { drop: [ALL] }
      volumes:
        - { name: conf, configMap: { name: lb-config } }
---
apiVersion: v1
kind: Service
metadata: { name: lb, namespace: demo, labels: { app: lb } }
spec:
  selector: { app: lb }
  ports:
    - { name: http,    port: 80,   targetPort: http }
    - { name: metrics, port: 9113, targetPort: metrics }
---
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: lb, namespace: demo }
spec:
  to: { kind: Service, name: lb }
  port: { targetPort: http }
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF

oc --context virt -n demo rollout status deploy/lb
echo "LB URL: https://$(oc --context virt -n demo get route lb -o jsonpath='{.spec.host}')"
```

Edge-TLS + Redirect on the Route means browsers that auto-upgrade to HTTPS
get a clean response (no 503 from the missing HTTPS terminator).

## 5. Virt: Prometheus that scrapes the LB exporter

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
      - job_name: nginx-lb
        static_configs:
          - targets: ['lb.demo.svc.cluster.local:9113']
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
```

## 6. Anaeem: ScaledObject scaling on the LB's rate

Assumes KEDA is already installed in `openshift-keda`.

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

## 7. Drive load through the LB

```bash
cat <<'EOF' | oc --context virt apply -f -
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

oc --context virt -n demo wait --for=condition=Ready pod/loadgen-pod --timeout=60s
```

Loadgen hits the **in-cluster LB Service** (skips the public Route, so it
doesn't go through the apps router for itself).

## 8. Watch the burst + the weighted distribution

In one shell, watch anaeem scaling:

```bash
watch -n 2 'oc --context anaeem -n demo get scaledobject demo-burst; \
  oc --context anaeem -n demo get hpa keda-hpa-demo-burst; \
  oc --context anaeem -n demo get pods -l app=prometheus-example-app'
```

In another, sample the LB output:

```bash
LB="https://$(oc --context virt -n demo get route lb -o jsonpath='{.spec.host}')"
for i in $(seq 1 60); do curl -sk "$LB/" | grep -oE 'from <b>[a-z]+</b>'; done | sort | uniq -c
```

Expected timeline:
1. Before anaeem has pods, every sample shows **virt** (LB falls back when
   the anaeem upstream returns 503).
2. Within ~30s KEDA marks the ScaledObject ACTIVE and scales anaeem 0→1→…→5.
3. Once anaeem pods are Ready, the same sampling shows roughly **5 anaeem
   for every 1 virt** (the weighting).
4. You can also open `$LB` in a browser — green page = anaeem, blue = virt.

## 9. Stop load + teardown

```bash
oc --context virt -n demo delete pod loadgen-pod --grace-period=0 --force

# Wait ~60s, anaeem scales back to 0; LB returns to all-virt.

# Full teardown:
oc --context anaeem -n demo delete scaledobject demo-burst
oc --context anaeem delete ns demo
oc --context virt   delete ns demo
```

## Troubleshooting

- **Every sample shows virt even after anaeem has pods.**
  The nginx LB caches upstream "down" state for `fail_timeout` (10s). Wait
  10s after anaeem pods become Ready, or `oc -n demo rollout restart deploy/lb`
  on virt to reset.

- **All requests hit the anaeem upstream get 503 from nginx LB.**
  Most often: `Host` header wrong. The OpenShift router on anaeem only
  matches on `Host: prometheus-example-app-demo.apps.anaeem.na-launch.com`.
  The `proxy_set_header Host` in the LB config must match the Route's
  `spec.host` exactly — no port, no port-number suffix from `$proxy_host`.

- **`Active=False` but loadgen is running.**
  Confirm the metric exists:
  `curl -sk "$PROM_URL/api/v1/query?query=sum(rate(nginx_http_requests_total[1m]))"`
  If empty, the exporter sidecar isn't scraping nginx's `/stub_status` — check
  `oc -n demo logs deploy/lb -c exporter`.

- **Browser loads `http://lb-demo...` and gets 503.**
  Browsers auto-upgrade to HTTPS. The LB Route must have `tls.termination:
  edge` with `insecureEdgeTerminationPolicy: Redirect` (step 4 has this).
