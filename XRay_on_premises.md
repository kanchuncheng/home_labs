#set up X-Ray daemon in a non-EKS Kubernetes environment and configure it with Istio for automatic instrumentation.



```yaml
# xray-k8s-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: xray-daemon
  namespace: default

---
# xray-k8s-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: xray-config
  namespace: default
data:
  config.yaml: |-
    # Local mode configuration for non-EKS setup
    TotalBufferSizeMB: 24
    Socket:
      UDPAddress: "0.0.0.0:2000"
    LocalMode: true
    LogLevel: "info"

---
# xray-k8s-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xray-daemon
  namespace: default
spec:
  selector:
    matchLabels:
      app: xray-daemon
  template:
    metadata:
      labels:
        app: xray-daemon
    spec:
      serviceAccountName: xray-daemon
      containers:
      - name: xray-daemon
        image: amazon/aws-xray-daemon:latest
        securityContext:
          privileged: false
        ports:
          - containerPort: 2000
            protocol: UDP
        volumeMounts:
          - name: config-volume
            mountPath: /etc/xray
      volumes:
        - name: config-volume
          configMap:
            name: xray-config

---
# istio-xray-injection.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: xray-tracing
  namespace: istio-system
spec:
  configPatches:
  - applyTo: BOOTSTRAP
    patch:
      operation: MERGE
      value:
        tracing:
          http:
            name: envoy.tracers.xray
            typed_config:
              "@type": type.googleapis.com/envoy.config.trace.v3.XRayConfig
              daemon_endpoint: "xray-daemon.default:2000"
              segment_name: "istio-proxy"

```

Here's how to set up X-Ray daemon in your home lab K8s cluster:

1. First, create an IAM user with the following minimal policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "xray:PutTraceSegments",
                "xray:PutTelemetryRecords",
                "xray:GetSamplingRules",
                "xray:GetSamplingTargets",
                "xray:GetSamplingStatisticSummaries"
            ],
            "Resource": "*"
        }
    ]
}
```

2. Set up AWS credentials in your cluster:
```bash
kubectl create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY \
  --from-literal=AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY \
  --from-literal=AWS_REGION=YOUR_REGION
```

3. Apply the configuration files from the artifact:
```bash
kubectl apply -f xray-k8s-serviceaccount.yaml
kubectl apply -f xray-k8s-configmap.yaml
kubectl apply -f xray-k8s-daemonset.yaml
kubectl apply -f istio-xray-injection.yaml
```

4. Verify the X-Ray daemon is running:
```bash
kubectl get pods -l app=xray-daemon
```

Key points about this setup:
- The daemon runs in local mode since we're not in EKS
- Istio is configured to automatically inject X-Ray tracing headers
- No application code changes are needed
- The daemon runs as a DaemonSet to ensure coverage across all nodes

Would you like me to explain any part of the configuration in more detail?
