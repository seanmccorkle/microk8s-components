# Overview

To test Cilium, as the Container Network Interface (CNI), and Traefik, as the Ingress Controller, use a simple web application that can demonstrate path routing, load balancing, and network policies. Both the `traefik/whoami` and `hashicorp/http-echo` containers can be utilized for testing though `traefik/whoami` will echo back HTTP request details (IPs, headers, hostname). Both make it easy to verify if Cilium and Traefik are working correctly. Another option, use `mendhak/http-https-echo` and configure port `8080`.

# Test Traefik

Identify the Traefik LoadBalancer external IP
```BASH
export TRAEFIK_IP=`kubectl get svc -n ingress -o jsonpath='{.items[].status.loadBalancer.ingress[0]}' | jq -r '.ip'`
```

Curl the endpoints

```BASH
curl http://$TRAEFIK_IP/v1 -> Should return details from an app-v1 pod
curl http://$TRAEFIK_IP/v2 -> Should return details from an app-v2 pod
```

# Test Cilium

## L7 Network Policy (Cilium CNI Feature)

By default, there are no restrictions between pod to pod communications. Use a `CiliumNetworkPolicy` to restrict traffic so only Traefik is allowed to talk to `app-v1`, block everything else.

```YAML
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-traefik-only
  namespace: network-test
spec:
  endpointSelector:
    matchLabels:
      app: app-v1
  ingress:
  - fromEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": ingress # Adjust to your Traefik namespace
        app.kubernetes.io/name: traefik
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

Curl the endpoint via Traefik will work because it is allowed
```BASH
curl http://$TRAEFIK_IP/v1
```

Curl the endpoint within the cluster and from a temporary pod will be denied

```BASH
kubectl run curl-debug -it --rm --restart=Never --image=alpine/curl -n network-test -- curl -m 5 -iv http://app-v1-service.network-test.svc.cluster.local # connection timeout
kubectl run curl-debug -it --rm --restart=Never --image=alpine/curl -n network-test -- curl -m 5 -iv http://app-v2-service.network-test.svc.cluster.local # connection established
```
