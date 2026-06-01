# Step-by-step manual setup

Same demo as `README.md`, but every `oc` command spelled out. No scripts.

## 0. Prereqs

- `oc` (4.14+) on your laptop.
- Cluster admin on both clusters.
- The two clusters have IP-level network connectivity to each other (in this
  lab it's the private 172.16.1.0/24).
- DNS for `*.apps.virt.na-launch.com` resolves to a reachable IP from inside
  anaeem pods. The lab's public DNS already returns the private 172.16.1.x
  IP for these names; if you fork the demo to other clusters, verify with
  `getent ahosts` from inside an anaeem pod.

## 1. Log in to both clusters and name the contexts

```bash
oc login -u <user> -p '...' https://api.virt.na-launch.com:6443
oc config rename-context $(oc config current-context) virt

oc login -u <user> -p '...' https://api.anaeem.na-launch.com:6443
oc config rename-context $(oc config current-context) anaeem
```

Now `oc --context virt …` and `oc --context anaeem …` target each cluster.

## 2. Virt: demo namespace + app + Service + Route

```bash
oc --context virt create ns demo

cat <<'EOF' | oc --context virt apply -f -
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
      containers:
        - name: app
          image: quay.io/brancz/prometheus-example-app:v0.5.0
          ports: [{ name: http, containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-example-app
  namespace: demo
  labels: { app: prometheus-example-app }
spec:
  selector: { app: prometheus-example-app }
  ports: [{ name: http, port: 8080, targetPort: http }]
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: prometheus-example-app
  namespace: demo
spec:
  to: { kind: Service, name: prometheus-example-app }
  port: { targetPort: http }
EOF

oc --context virt -n demo rollout status deploy/prometheus-example-app
```

## 3. Virt: dedicated Prometheus + edge-TLS Route

This is the metrics source KEDA on anaeem will poll.

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
      - job_name: prometheus-example-app
        static_configs:
          - targets: ['prometheus-example-app:8080']
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
metadata: { name: prometheus, namespace: demo, labels: { app: prometheus } }
spec:
  to: { kind: Service, name: prometheus }
  port: { targetPort: web }
  tls: { termination: edge, insecureEdgeTerminationPolicy: Redirect }
EOF

oc --context virt -n demo rollout status deploy/prometheus

PROM_URL="https://$(oc --context virt -n demo get route prometheus -o jsonpath='{.spec.host}')"
echo "PROM_URL=$PROM_URL"
```

Keep `$PROM_URL` in your shell — you'll paste it into the anaeem ScaledObject.

Sanity check from your laptop (no token):

```bash
# Hit the app a few times so the rate is non-zero, then query
APP_HOST=$(oc --context virt -n demo get route prometheus-example-app -o jsonpath='{.spec.host}')
for i in $(seq 1 5); do curl -s "http://$APP_HOST/" >/dev/null; done
sleep 20
curl -sk "$PROM_URL/api/v1/query" --data-urlencode 'query=sum(rate(http_requests_total[1m]))' | python3 -m json.tool
```

You should see a non-empty `result` array.

## 4. Anaeem: install the Custom Metrics Autoscaler operator (KEDA)

Skip if `oc --context anaeem get csv -n openshift-keda | grep custom-metrics`
already shows `Succeeded`.

```bash
cat <<'EOF' | oc --context anaeem apply -f -
apiVersion: v1
kind: Namespace
metadata: { name: openshift-keda, labels: { openshift.io/cluster-monitoring: "true" } }
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata: { name: openshift-keda, namespace: openshift-keda }
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-custom-metrics-autoscaler-operator
  namespace: openshift-keda
spec:
  channel: stable
  installPlanApproval: Automatic
  name: openshift-custom-metrics-autoscaler-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for the CSV to reach Succeeded (~1–3 min)
oc --context anaeem -n openshift-keda get csv -w
```

Once `Succeeded`:

```bash
cat <<'EOF' | oc --context anaeem apply -f -
apiVersion: keda.sh/v1alpha1
kind: KedaController
metadata: { name: keda, namespace: openshift-keda }
spec:
  watchNamespace: ""
  operator: { logLevel: info, logEncoder: console }
  metricsServer: { logLevel: "0" }
  serviceAccount: {}
EOF

oc --context anaeem -n openshift-keda wait \
  --for=condition=Available deploy/keda-operator deploy/keda-metrics-apiserver \
  --timeout=300s
```

## 5. Anaeem: demo namespace + burst Deployment (replicas: 0)

```bash
oc --context anaeem create ns demo

cat <<'EOF' | oc --context anaeem apply -f -
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
      containers:
        - name: app
          image: quay.io/brancz/prometheus-example-app:v0.5.0
          ports: [{ name: http, containerPort: 8080 }]
EOF
```

## 6. Anaeem: ScaledObject pointed at virt's Prometheus Route

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
        query: sum(rate(http_requests_total[1m]))
        threshold: "5"
        activationThreshold: "2"
        unsafeSsl: "true"
EOF

oc --context anaeem -n demo get scaledobject demo-burst
```

No `authenticationRef`, no `authModes`, no Secret, no SA token.

## 7. Drive load on virt

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
          end=$(( $(date +%s) + 600 ))
          while [ "$(date +%s)" -lt "$end" ]; do
            for i in $(seq 1 30); do
              curl -s -o /dev/null http://prometheus-example-app.demo.svc:8080/ &
            done
            wait
          done
EOF

oc --context virt -n demo wait --for=condition=Ready pod/loadgen-pod --timeout=60s
```

(Plain Pod, not a Job — a misbehaving admission webhook on virt blocks the
Job controller from creating pods in this namespace. Plain Pods are fine.)

## 8. Watch the burst

In one shell, virt's rate (no token, just curl):

```bash
watch -n 5 "curl -sk '$PROM_URL/api/v1/query' \
  --data-urlencode 'query=sum(rate(http_requests_total[1m]))' \
  | python3 -c 'import sys,json; r=json.load(sys.stdin)[\"data\"][\"result\"]; print(float(r[0][\"value\"][1]) if r else 0)'"
```

In another, anaeem scaling:

```bash
watch -n 2 'oc --context anaeem -n demo get scaledobject demo-burst; \
  oc --context anaeem -n demo get hpa keda-hpa-demo-burst; \
  oc --context anaeem -n demo get pods -l app=prometheus-example-app'
```

Expected: `ACTIVE=True` within a poll cycle, HPA appears, deployment 0 → 5.

## 9. Stop load, watch scale-down

```bash
oc --context virt -n demo delete pod loadgen-pod --grace-period=0 --force
```

After ~60s (`cooldownPeriod`), `ACTIVE=False` and the deployment scales to 0.

## 10. Teardown

```bash
oc --context anaeem -n demo delete scaledobject demo-burst
oc --context anaeem delete ns demo

oc --context virt delete ns demo
```

(Leave UWM and KEDA themselves alone unless you want a full wipe.)

## Troubleshooting

- **`ScaledObject` `READY=False` with `context deadline exceeded`**
  KEDA can't reach the virt Prometheus Route. From an `openshift-keda` pod:
  ```bash
  oc --context anaeem -n openshift-keda debug node/<node> -- chroot /host \
    curl -ksv --max-time 8 "$PROM_URL/api/v1/query?query=up"
  ```
  TCP timeout → network. Wrong IP → DNS.

- **HPA shows `<unknown>/5`**
  `keda-metrics-apiserver` can't reach `keda-operator` over gRPC. Restart:
  ```bash
  oc --context anaeem -n openshift-keda rollout restart \
    deploy/keda-operator deploy/keda-metrics-apiserver
  ```

- **Empty `result` from the query**
  Prometheus hasn't scraped yet (wait 30s after deploying) or the target is
  down: `curl -sk "$PROM_URL/api/v1/targets"` and check `health`.
