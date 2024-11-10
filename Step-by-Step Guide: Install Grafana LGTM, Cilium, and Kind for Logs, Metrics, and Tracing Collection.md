### Step-by-Step Guide: Install Grafana LGTM, Cilium, and Kind for Logs, Metrics, and Tracing Collection

---

### **Prerequisites**
1. **System Requirements**:
   - Linux/macOS/Windows with Docker installed.
   - At least 4GB of RAM and 2 CPUs for running `kind` clusters.
2. **Install Required Tools**:
   - **Kind**: Install Kind ([official guide](https://kind.sigs.k8s.io/)).
   - **kubectl**: Install kubectl ([official guide](https://kubernetes.io/docs/tasks/tools/)).
   - **Helm**: Install Helm ([official guide](https://helm.sh/docs/intro/install/)).
   - **Grafana CLI**: Pre-installed with Grafana binary distributions.

---

### **1. Set Up Kind Cluster**
1. Create a Kind cluster with the following configuration:
   ```bash
   cat <<EOF | kind create cluster --name lgtm-cilium --config=-
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
   - role: control-plane
     extraPortMappings:
     - containerPort: 30000
       hostPort: 30000
       protocol: TCP
   EOF
   ```
   - This opens port 30000 for accessing Grafana.

2. Verify the cluster:
   ```bash
   kubectl cluster-info
   ```

---

### **2. Install Cilium for Networking and Observability**
Cilium acts as a CNI and provides observability features like Hubble for metrics and tracing.

1. Add the Cilium Helm repo:
   ```bash
   helm repo add cilium https://helm.cilium.io/
   helm repo update
   ```

2. Install Cilium with Hubble and Prometheus enabled:
   ```bash
   helm install cilium cilium/cilium \
       --namespace kube-system \
       --set kubeProxyReplacement=strict \
       --set hubble.enabled=true \
       --set hubble.relay.enabled=true \
       --set hubble.ui.enabled=true \
       --set operator.prometheus.enabled=true \
       --set prometheus.enabled=true
   ```

3. Verify the Cilium installation:
   ```bash
   cilium status
   ```

4. Forward the Hubble UI port:
   ```bash
   kubectl port-forward -n kube-system service/hubble-ui 12000:80
   ```
   Access Hubble UI at `http://localhost:12000`.

---

### **3. Install Grafana Loki, Prometheus, and Tempo (LGTM Stack)**
Grafana's LGTM stack provides logging (Loki), metrics (Prometheus), and tracing (Tempo).

#### Install Prometheus
1. Add the Prometheus Helm repo:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Install Prometheus:
   ```bash
   helm install prometheus prometheus-community/prometheus \
       --namespace monitoring \
       --create-namespace
   ```

3. Verify Prometheus:
   ```bash
   kubectl get pods -n monitoring
   ```

---

#### Install Loki
1. Add the Grafana Helm repo:
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```

2. Install Loki:
   ```bash
   helm install loki grafana/loki-stack \
       --namespace monitoring \
       --set grafana.enabled=false \
       --set promtail.enabled=true
   ```

---

#### Install Tempo
1. Install Tempo:
   ```bash
   helm install tempo grafana/tempo \
       --namespace monitoring
   ```

2. Verify Tempo:
   ```bash
   kubectl get pods -n monitoring
   ```

---

### **4. Install Grafana**
1. Install Grafana:
   ```bash
   helm install grafana grafana/grafana \
       --namespace monitoring \
       --set adminPassword='admin' \
       --set service.type=NodePort \
       --set service.nodePort=30000 \
       --set datasources.prometheus.url=http://prometheus-server.monitoring.svc.cluster.local
   ```

2. Retrieve the Grafana admin password:
   ```bash
   kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```

3. Access Grafana at `http://localhost:30000`. Log in with `admin` and the retrieved password.

---

### **5. Integrate Data Sources in Grafana**
1. **Add Loki as a data source:**
   - In Grafana, go to **Configuration > Data Sources**.
   - Add Loki using the URL `http://loki.monitoring.svc.cluster.local`.

2. **Add Tempo as a data source:**
   - Add Tempo using the URL `http://tempo.monitoring.svc.cluster.local`.

3. **Import Dashboards:**
   - Import Cilium, Prometheus, Loki, and Tempo dashboards available on the Grafana website or Helm values.

---

### **6. Verify Logging, Metrics, and Tracing**
1. **Metrics**:
   - Visit the Prometheus dashboard or Grafana Prometheus metrics dashboards.
2. **Logs**:
   - Query logs in Grafana using Loki.
3. **Tracing**:
   - View traces in Grafana through Tempo.

---

### **7. (Optional) Enable Hubble Observability in Grafana**
1. Add Hubble as a data source in Grafana:
   - Use the URL `http://hubble-relay.kube-system.svc.cluster.local`.

2. Import Hubble dashboards from the Cilium GitHub repository.

---

You now have a fully operational LGTM stack with Cilium in a Kind cluster for logs, metrics, and tracing collection! Let me know if you need further assistance.
