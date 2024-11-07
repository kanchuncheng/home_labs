Setting up **AWS X-Ray** in a **Kubernetes (K8s)** environment in your **home lab** for **auto-instrumentation** using **access keys** without modifying your application code can be achieved with a few straightforward steps. The general approach is to deploy the **X-Ray Daemon** in your Kubernetes cluster, configure your pods to send trace data to the daemon, and use **AWS access keys** for authentication.

The main components for this setup are:

1. **X-Ray Daemon (running as a DaemonSet)**
2. **Auto-instrumentation of AWS SDK calls** (handled by the AWS X-Ray SDK)
3. **IAM Access Keys for authentication**

Here are the steps to get AWS X-Ray running in a K8s cluster for auto-instrumentation, without requiring changes to your application code:

---

### Step 1: Deploy the X-Ray Daemon in Kubernetes

The X-Ray Daemon collects trace data from your application and sends it to AWS X-Ray. It runs on every node in your Kubernetes cluster and listens on UDP port 2000. You will deploy this Daemon as a **DaemonSet** so that the X-Ray Daemon runs on every node in the cluster.

#### 1.1 Create the DaemonSet YAML

Create a file called `xray-daemonset.yaml` to define the DaemonSet for the X-Ray Daemon:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xray-daemonset
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
      containers:
        - name: xray-daemon
          image: amazon/aws-xray-daemon:latest
          ports:
            - containerPort: 2000
              protocol: UDP
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
      nodeSelector:
        kubernetes.io/os: linux
```

This YAML file does the following:
- **DaemonSet** ensures the X-Ray Daemon runs on each node in the cluster.
- The X-Ray Daemon listens on **UDP port 2000** by default to receive trace data.
- The `nodeSelector` ensures it runs on Linux nodes (adjust as needed).

#### 1.2 Apply the DaemonSet to Kubernetes

Now, apply the `xray-daemonset.yaml` to your Kubernetes cluster using `kubectl`:

```bash
kubectl apply -f xray-daemonset.yaml
```

This command will deploy the X-Ray Daemon on each node in your Kubernetes cluster.

---

### Step 2: Set Up AWS Access Keys for Authentication

AWS X-Ray requires authentication to send trace data to the X-Ray service. In a home lab setup, you'll typically use **IAM access keys** for authentication.

#### 2.1 Create AWS Access Keys

1. **Create IAM User with X-Ray Permissions**:
   - Go to the **AWS IAM console**.
   - Create a new user or use an existing user.
   - Attach the `AWSXRayDaemonWriteAccess` policy to this user to allow sending trace data to AWS X-Ray.

2. **Create Access Keys**:
   - In the IAM console, go to the user’s **Security credentials** tab.
   - Create a new **Access Key** and download the key pair (Access Key ID and Secret Access Key).

#### 2.2 Set the AWS Credentials in Kubernetes

You can securely pass your AWS credentials to your Kubernetes pods using a **Kubernetes Secret**.

1. Create a Kubernetes Secret to store your AWS access keys:

   ```bash
   kubectl create secret generic aws-xray-credentials \
     --from-literal=AWS_ACCESS_KEY_ID=your-access-key-id \
     --from-literal=AWS_SECRET_ACCESS_KEY=your-secret-access-key \
     --from-literal=AWS_DEFAULT_REGION=us-east-1  # Replace with your region
   ```

2. Verify the secret has been created:

   ```bash
   kubectl get secret aws-xray-credentials -o yaml
   ```

---

### Step 3: Configure the Kubernetes Pods for Auto-Instrumentation

Now, configure your application pods to interact with the X-Ray Daemon and use the AWS credentials.

#### 3.1 Modify Your Pod Deployment to Use the Secret

In the `Deployment` or `Pod` configuration of your application, you'll need to mount the secret containing your AWS credentials and ensure the application is set up to communicate with the X-Ray Daemon.

Here's an example of how to modify your **Pod specification** to use the secret:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app-image:latest  # Replace with your app's image
          env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-xray-credentials
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-xray-credentials
                  key: AWS_SECRET_ACCESS_KEY
            - name: AWS_DEFAULT_REGION
              valueFrom:
                secretKeyRef:
                  name: aws-xray-credentials
                  key: AWS_DEFAULT_REGION
          volumeMounts:
            - name: xray-socket
              mountPath: /var/run/xray
      volumes:
        - name: xray-socket
          emptyDir: {}
```

This YAML does the following:
- The `env` section injects the AWS credentials (from the `aws-xray-credentials` secret) into the environment variables of the container.
- The X-Ray Daemon is typically reachable via **UDP port 2000** on the `xray-daemon` service (see next step).
- The `AWS_DEFAULT_REGION` environment variable is set to the region where your X-Ray service is located (e.g., `us-east-1`).

#### 3.2 Enable AWS SDK Auto-Instrumentation

Since you don’t want to modify your application code, AWS SDK-based auto-instrumentation will happen automatically if your application uses AWS SDKs (e.g., for S3, DynamoDB, etc.). The **AWS SDK** is already integrated with the X-Ray SDK, and the X-Ray Daemon will automatically handle the tracing of these calls.

No code changes are needed in your application. Just ensure the application is configured to use AWS SDKs and sends requests to AWS services.

---

### Step 4: Monitor Traces in AWS X-Ray

Once your X-Ray Daemon is running and your pods are configured to send trace data, you can monitor the trace data in the **AWS X-Ray Console**.

1. **Access the X-Ray Console**:
   - Go to **AWS X-Ray Console** in your AWS account.
   - You should start seeing traces from your application after the X-Ray Daemon has forwarded the data.

2. **Explore Traces**:
   - In the console, explore trace data, performance bottlenecks, and service maps. AWS X-Ray will provide insights into which AWS services your application is interacting with and how long each request takes.

---

### Step 5: Troubleshooting and Validation

If you're not seeing trace data in X-Ray, check the following:

1. **Verify the DaemonSet**: Ensure the X-Ray Daemon is running on all nodes and is not encountering any issues. Use `kubectl get pods -l app=xray-daemon` to check the daemon's status.
2. **Check Logs**: Look at the logs of the X-Ray Daemon pod for any errors:
   ```bash
   kubectl logs -l app=xray-daemon
   ```
3. **Check Pod Connectivity**: Ensure your application pods can reach the X-Ray Daemon over **UDP port 2000**.
4. **Ensure Proper IAM Permissions**: Ensure your access keys have the correct IAM permissions (specifically, `AWSXRayDaemonWriteAccess`).

---

### Summary

To set up **AWS X-Ray for auto-instrumentation** in your **Kubernetes (K8s)** home lab environment with **access keys** and **no code changes**:

1. **Deploy the X-Ray Daemon** in your Kubernetes cluster using a **DaemonSet**.
2. **Create AWS IAM access keys** and store them in a **Kubernetes Secret**.
3. **Configure your application pods** to use the X-Ray Daemon by mounting the AWS credentials and setting the environment variables.
4. **Monitor traces** in the AWS X-Ray console.

With this setup, the X-Ray SDK will automatically trace any **AWS SDK calls** your application makes without needing changes to the application code itself.
