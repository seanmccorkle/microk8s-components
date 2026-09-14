# Overview

To test Cilium, as the Container Network Interface (CNI), and Traefik, as the Ingress Controller, use a simple web application that can demonstrate path routing, load balancing, and network policies. Both the `traefik/whoami` and `hashicorp/http-echo` containers echo back HTTP request details (IPs, headers, hostname) making it easy to verify if Cilium and Traefik are working correctly.

# Test Traefik

Identify the Traefik LoadBalancer external IP
```BASH
kubectl get svc -n traefik
```

Curl the endpoints

```BASH
curl http://<TRAEFIK_IP>/v1 -> Should return details from an app-v1 pod
curl http://<TRAEFIK_IP>/v2 -> Should return details from an app-v2 pod
```

# Test Cilium

## L7 Network Policy (Cilium CNI Feature)

By default, there are no restrictions between pod to pod communications. Use a `CiliumNetworkPolicy` to restrict traffic so only Traefik is allowed to talk to `app-v1`, block everything else.

```YAML
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-traefik-only
spec:
  endpointSelector:
    matchLabels:
      app: app-v1
  ingress:
  - fromEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": traefik # Adjust to your Traefik namespace
        app.kubernetes.io/name: traefik
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

Curl the endpoint via Traefik will work because it is allowed
```BASH
curl http://<TRAEFIK_IP>/v1
```

Identify pod or service IP address for `app-v1` and attempt to curl directly which will fail as Cilium will drop the packets at the eBPF layer
