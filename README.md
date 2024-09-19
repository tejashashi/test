Issue1: Apphealth of argocd is in Degraded state and carina-instance replicaset,deployment are degraded because there is no service account in deployment ns(default)

The error message encountering is:
The error message you are encountering:

```
Error creating: pods "carina-instance-5d4c788757-" is forbidden: error looking up service account default/argocd-server: serviceaccount "argocd-server" not found
```

indicates that Kubernetes is trying to create a pod that is bound to a service account named `argocd-server` in the `default` namespace, but it cannot find this service account. Here's how you can resolve it:

### Steps to Fix

1. **Verify Namespace:**
   Check whether the service account is being referenced in the correct namespace. If you're deploying ArgoCD in a different namespace (e.g., `argocd`), ensure that the correct namespace is being used when creating the pods.

   You can check the current namespace for your pod deployment using:
   ```bash
   kubectl get pods --all-namespaces | grep carina-instance
   ```

2. **Check if the Service Account Exists:**
   Confirm whether the service account `argocd-server` exists in the namespace where you're trying to create the pod:
   ```bash
   kubectl get serviceaccount argocd-server -n <namespace>
   ```

   If it doesn’t exist, it’s likely that the ArgoCD installation did not create the necessary service account.

3. **Create the Service Account:**
   If the service account is missing, you can create it manually:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
 	name: argocd-server
 	namespace: <your-namespace>
   ```

   Apply this file using:
   ```bash
   kubectl apply -f serviceaccount.yaml
   ```

4. **Reinstall ArgoCD (if needed):**
   If the service account still isn't being created, try reinstalling ArgoCD properly in the correct namespace:
   ```bash
   kubectl apply -n <your-namespace> -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

   This should create all the necessary resources, including the `argocd-server` service account.

5. **Check RBAC Permissions:**
   If the service account exists but the error persists, verify that the service account has the appropriate RoleBindings or ClusterRoleBindings for the operations it needs to perform. You can list the role bindings associated with the `argocd-server` service account like this:
   ```bash
   kubectl get rolebinding,clusterrolebinding -n <namespace> | grep argocd-server
   ```

6. **Redeploy the Pod:**
   Once the service account issue is resolved, redeploy the pod to see if the error is fixed.

**How you can redeploy a pod**
To redeploy a pod in Kubernetes, you generally need to delete the existing pod and let the associated deployment or other controller (like a ReplicaSet, StatefulSet, or DaemonSet) recreate it. Here's how you can redeploy a pod, depending on how the pod was created:

### 1. **If the Pod is Managed by a Deployment (Most Common Case)**

#### Option 1: **Scale the Deployment Down and Up**
You can scale the deployment down to 0 and then scale it back up. This will terminate the current pod(s) and start new ones.

```bash
# Scale down to 0
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>

# Scale back up to the original number of replicas
kubectl scale deployment <deployment-name> --replicas=<original-replicas> -n <namespace>
```

#### Option 2: **Rollout Restart (Kubernetes 1.15+)**
If you're using Kubernetes version 1.15 or higher, you can use the following command to restart the deployment, which will trigger the recreation of the pods:

```bash
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

This command tells Kubernetes to restart the deployment, which will kill the current pods and start new ones with the same configuration.

### 2. **If the Pod is Managed by a StatefulSet**

StatefulSets ensure ordered and stable pod identities. To redeploy a pod managed by a StatefulSet:

```bash
kubectl rollout restart statefulset <statefulset-name> -n <namespace>
```

This will recreate the pods in the StatefulSet.

### 3. **If the Pod is Managed by a DaemonSet**

DaemonSets ensure that a copy of a pod runs on each node. To redeploy pods managed by a DaemonSet:

```bash
kubectl rollout restart daemonset <daemonset-name> -n <namespace>
```

### 4. **If the Pod is Not Managed by a Controller (Standalone Pod)**

If the pod is a standalone pod (not managed by a Deployment, ReplicaSet, etc.), you need to delete it manually. Kubernetes will not recreate it unless it's part of a controller.

```bash
kubectl delete pod <pod-name> -n <namespace>
```

You'll then need to apply the pod's manifest file again or create the pod manually if it’s not managed by any controller. Example:

```bash
kubectl apply -f pod.yaml
```

### 5. **Verify the Pod Status**
Once you've triggered a redeployment using any of the methods above, you can verify the status of the new pod using:

```bash
kubectl get pods -n <namespace>
```

You should see the new pod coming up with a new creation timestamp.

---





