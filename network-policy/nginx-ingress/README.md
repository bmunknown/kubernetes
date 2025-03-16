NGINX Ingress Controller is responsible for routing external traffic into the Kubernetes cluster based on the rules you define, so it's important to ensure secure and controlled access.

### 1. **Identify NGINX Ingress Controller Components**
NGINX Ingress Controller typically consists of:
- **NGINX Ingress Controller Pods**: These pods run the NGINX proxy and manage traffic routing.
- **NGINX Ingress Controller Service**: Exposes the NGINX Ingress Controller to the external network.
- **Ingress Resources**: Define how external HTTP/S traffic is routed to the internal services.

### 2. **Create Network Policies for NGINX Ingress Controller**

You’ll need to define Network Policies to control traffic within the cluster, ensuring that only authorized services and pods can communicate with the Ingress Controller and vice versa.

#### **Network Policy for NGINX Ingress Controller Pods**
This example policy allows the NGINX Ingress Controller pods to communicate with each other within the same namespace (typically the `ingress-nginx` namespace), but restricts access from other namespaces unless specifically allowed.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-nginx
  namespace: ingress-nginx
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80   # HTTP Port
        - protocol: TCP
          port: 443  # HTTPS Port
```

- **namespace: ingress-nginx**: Specifies the namespace where NGINX Ingress Controller is deployed.
- **podSelector**: This selects the NGINX Ingress Controller pods based on the label `app.kubernetes.io/name: ingress-nginx`.
- **ingress**: Defines which pods can send traffic to the NGINX Ingress Controller. In this case, only NGINX Ingress Controller pods can communicate with each other.
- **ports**: Allows access to the NGINX Ingress Controller on HTTP (80) and HTTPS (443).

#### **Network Policy for Access to NGINX Ingress Controller Service**
If you want to control access to the NGINX Ingress Controller service (which is exposed to external traffic), you can create a policy that allows traffic only from specific sources (e.g., trusted IPs or specific pods).

Here’s an example of allowing only certain external IPs to access the Ingress Controller service:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-access-to-ingress-nginx
  namespace: ingress-nginx
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  ingress:
    - from:
        - ipBlock:
            cidr: 203.0.113.0/24  # Allowed external IP range
      ports:
        - protocol: TCP
          port: 80   # HTTP Port
        - protocol: TCP
          port: 443  # HTTPS Port
```

- **ipBlock**: Defines the allowed external IP range that can access the Ingress Controller.
- **ports**: Allows access on HTTP (80) and HTTPS (443) ports.

#### **Network Policy for Internal Access to NGINX Ingress**
By default, the NGINX Ingress Controller should be able to route traffic to the appropriate backend services. Here’s a policy that restricts access to the Ingress Controller from other services unless they are specifically allowed.

For instance, you can allow only certain namespaces (e.g., `default` or `my-app-namespace`) to send traffic to the Ingress Controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-namespaces-to-ingress-nginx
  namespace: ingress-nginx
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: default   # Allow traffic from the 'default' namespace
        - namespaceSelector:
            matchLabels:
              name: my-app-namespace   # Allow traffic from the 'my-app-namespace'
      ports:
        - protocol: TCP
          port: 80  # HTTP Port
        - protocol: TCP
          port: 443  # HTTPS Port
```

- **namespaceSelector**: Allows traffic from pods within specific namespaces (`default`, `my-app-namespace`) to communicate with the NGINX Ingress Controller.
- **ports**: Again, only HTTP and HTTPS ports are allowed.

### 3. **Network Policy for External Services Accessing NGINX Ingress**
If you are using an external load balancer or an external service to access the NGINX Ingress Controller, you can restrict external services by applying an `ipBlock` rule. 

Here's an example that restricts access to only trusted external IPs:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-external-access-to-ingress-nginx
  namespace: ingress-nginx
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  ingress:
    - from:
        - ipBlock:
            cidr: 198.51.100.0/24  # Trusted external IP range
      ports:
        - protocol: TCP
          port: 80  # HTTP Port
        - protocol: TCP
          port: 443  # HTTPS Port
```

- **ipBlock**: Specifies the allowed external IP range that can access the NGINX Ingress Controller's services.
- **ports**: Restricts external access to HTTP (80) and HTTPS (443) ports only.

### 4. **Testing and Validation**
After applying these Network Policies, you should:
- Ensure that NGINX Ingress Controller pods can still communicate with each other.
- Verify that only the allowed external IPs or services can access the Ingress Controller service.
- Check that only specific internal services or namespaces can route traffic through the Ingress Controller.

You can use `kubectl describe networkpolicy` to verify the status of the applied policies.

---

### Summary:
- **Network Policy for NGINX Ingress Controller Pods**: Allows traffic between Ingress Controller pods in the same namespace.
- **Network Policy for External Access to NGINX Ingress**: Controls which external IPs or services can access the Ingress Controller.
- **Network Policy for Internal Access to NGINX Ingress**: Restricts which internal namespaces or services can communicate with the Ingress Controller.

