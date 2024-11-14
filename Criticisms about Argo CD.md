While Argo CD is a powerful and popular GitOps tool, it's not without its criticisms. Here are some common points of contention:

**Complexity:**

* **Steep Learning Curve:** Some users find the initial setup and configuration of Argo CD to be complex and time-consuming.
* **Complex Workflows:** Implementing advanced workflows, such as canary deployments or A/B testing, can require significant effort and configuration.

**User Interface:**

* **Limited Functionality:** While the web UI is user-friendly for basic operations, it may lack advanced features and customization options.
* **Performance Issues:** In large-scale deployments, the UI can become slow and unresponsive.

**Synchronization and Reconciliation:**

* **Delayed Synchronization:** In some cases, Argo CD may experience delays in synchronizing applications with the Git repository.
* **Aggressive Reconciliation:** The default reconciliation behavior can sometimes lead to unnecessary deployments and resource churn.

**Security:**

* **Exposure Risk:** If not configured properly, Argo CD can be exposed to security risks.
* **Permissions and Access Control:** Managing permissions and access control can be complex, especially in large-scale environments.

**Others**
* **Too many CRD:** Users can't see pods, instead of applications. This changes the architechure that users are used to in a native k8s.
* **Multiple tentants are not necessary in most of enterprises:**
* **Too fat:** Not just too complex, especially when HA function is used.
* **Polls to github:** The interval is 3 minutes, not realtime, kind of slow.
* **Canary deployment can be done by Ingress:**
* **Blue/green, rolling update deployment can be done by Service:** This way is more intuitive and effective.