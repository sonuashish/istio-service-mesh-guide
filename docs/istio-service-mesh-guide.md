# Istio and Service Mesh on Kubernetes

> A practical field guide to service-to-service networking, Envoy sidecars, workload mTLS, traffic shifting, admission webhooks, Bookinfo and troubleshooting.

![Istio service mesh cover](assets/slides/slide-01.png)

## What you'll learn

- What a service mesh does and where it fits in Kubernetes networking
- North-south vs east-west traffic
- How Istio's Envoy data plane and `istiod` control plane work
- How automatic sidecar injection uses a mutating admission webhook
- How workload mTLS is configured with `PeerAuthentication`
- How `DestinationRule` and `VirtualService` implement traffic splitting
- How to explore the mesh with Bookinfo and Kiali
- How to troubleshoot common configuration problems

## Contents

1. [Service mesh fundamentals](#1-service-mesh-fundamentals)
2. [Why Istio?](#2-why-istio)
3. [Architecture: data plane and control plane](#3-architecture-data-plane-and-control-plane)
4. [How sidecar injection works](#4-how-sidecar-injection-works)
5. [Install Istio](#5-install-istio)
6. [Deploy Bookinfo](#6-deploy-bookinfo)
7. [Secure workloads with mTLS](#7-secure-workloads-with-mtls)
8. [Manage traffic and roll out a canary](#8-manage-traffic-and-roll-out-a-canary)
9. [Observability with Kiali](#9-observability-with-kiali)
10. [Gateway vs Kubernetes Ingress](#10-gateway-vs-kubernetes-ingress)
11. [Troubleshooting](#11-troubleshooting)
12. [Command cheat sheet](#12-command-cheat-sheet)
13. [Interview questions](#13-interview-questions)
14. [Further considerations](#14-further-considerations)

## Key terms at a glance

| Term | Meaning |
|---|---|
| Service mesh | A layer for managing and securing service-to-service traffic |
| Istio | A service mesh platform commonly used with Kubernetes |
| Envoy sidecar | A proxy container alongside an application container in sidecar mode |
| `istiod` | Istio control-plane component that distributes configuration and manages workload identity |
| mTLS | Mutual TLS: both peers authenticate using certificates and establish an encrypted connection |
| `VirtualService` | Istio routing rules, including traffic weights |
| `DestinationRule` | Destination policies and named subsets such as `v1` and `v2` |
| `PeerAuthentication` | Policy for inbound workload mTLS |
| Gateway | Istio configuration for traffic entering or leaving the mesh |
| Kiali | Console for visualising service mesh topology and health |
| Admission webhook | An API-server callback that can mutate or validate matching resource requests |

## 1. Service mesh fundamentals

A service mesh manages communication between services. In a microservices application, one request can travel through several workloads: a login service calls a catalog service, which calls payments, which may call notifications.

### North-south vs east-west traffic

- **North-south:** traffic entering or leaving the cluster, for example a user reaching an application through a load balancer or ingress gateway.
- **East-west:** traffic between services inside the cluster, for example catalog calling payments.

A service mesh is primarily used to manage east-west traffic, while it can also provide a consistent model for ingress and egress.

![North-south and east-west traffic](assets/slides/slide-02.png)

## 2. Why Istio?

Kubernetes Services already let workloads discover and reach one another. Istio adds capabilities around those connections:

| Capability | What it provides |
|---|---|
| Workload security | Authenticated, encrypted service-to-service communication with mTLS |
| Traffic management | Weighted routing, canary releases, timeouts, retries and circuit breaking |
| Observability | Telemetry that can be visualised with tools such as Kiali |

**The trade-off:** proxies consume CPU and memory, add some latency, and increase operational complexity. A mesh is useful when its security, traffic and observability features justify that cost.

### TLS vs mTLS

In ordinary TLS, the client validates the server's certificate. In mutual TLS, both peers authenticate each other using certificates. Istio can manage workload identity and certificates so application code does not need to implement this logic itself.

### Canary release in brief

Suppose a service has `v1` and a new `v2`. Route a small percentage of requests to `v2`, observe health and business metrics, then gradually increase its share. If the new version behaves well, promote it to 100%; otherwise, roll traffic back to the stable version.

## 3. Architecture: data plane and control plane

In Istio's **sidecar mode**, an Envoy proxy runs alongside each application container in an injected Pod. Traffic entering or leaving the workload is intercepted by the proxy. The proxies form the data plane.

`istiod` is the control plane. It distributes configuration and manages workload certificates/identity. The proxies enforce the configuration on the traffic path.

![Sidecar architecture](assets/slides/slide-04.png)

A service-to-service request follows this simplified path:

1. The source application sends a request.
2. The source Envoy proxy intercepts it and applies the configured policy.
3. The destination Envoy proxy receives the connection and enforces inbound policy.
4. The request reaches the destination application.

The application containers do not need to implement Istio routing or mTLS logic.

## 4. How sidecar injection works

Automatic injection uses Kubernetes admission control.

A simplified Pod-create flow:

1. A client submits a Pod create request to the Kubernetes API server.
2. Authentication and authorization are performed.
3. Admission controllers process the request. Mutating admission runs before validating admission.
4. If the request matches Istio's injection webhook configuration, the API server calls the webhook.
5. The webhook returns a modified Pod specification that includes the proxy container and related configuration.
6. The admitted object is stored by the API server.

The built-in `MutatingAdmissionWebhook` controller invokes configured external webhooks; the webhook configuration determines which requests match and where the callback is sent.

### Enable automatic injection

```bash
kubectl label namespace default istio-injection=enabled
```

This affects newly created Pods in the labelled namespace. Existing Pods are not retroactively modified; recreate or restart the workloads after enabling injection.

Verify:

```bash
kubectl get namespace default --show-labels
kubectl get pods
kubectl describe pod <pod-name>
```

In a sidecar-injected Pod, `kubectl get pods` commonly shows `2/2` containers ready: the application and the proxy.

### Admission control: mutation vs validation

- **Mutation:** an admission plugin or webhook changes the incoming object before it is persisted. For example, a defaulting mechanism can populate a missing field.
- **Validation:** an admission plugin or webhook checks the object and can reject it if a policy is violated.

Useful commands:

```bash
kubectl get mutatingwebhookconfiguration
kubectl get validatingwebhookconfiguration
```

The exact built-in admission plugins enabled depend on Kubernetes distribution and API-server configuration. Avoid assuming every cluster has an identical list.

## 5. Install Istio

You need a working Kubernetes cluster and `kubectl`. Install a version that is supported for your Kubernetes environment; check the official Istio installation guide before choosing a version.

The following mirrors the versioned download pattern from the original lab notes. `1.31.1` is an example version, not a recommendation to use it indefinitely:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.31.1 sh -
cd istio-*
export PATH="$PWD/bin:$PATH"

istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
```

The demo profile is intended for learning and evaluation, not as a production baseline. The installation includes `istiod` and an ingress gateway.

Check the installation:

```bash
istioctl version
kubectl get pods -n istio-system
istioctl analyze
```

## 6. Deploy Bookinfo

Bookinfo is Istio's sample application. It is useful because it contains several services and multiple versions of the reviews service.

| Service | Language in sample | Role |
|---|---|---|
| `productpage` | Python | User-facing page; calls details and reviews |
| `details` | Ruby | Product details |
| `reviews` | Java | Reviews; has v1, v2 and v3 variants |
| `ratings` | Node.js | Star ratings; called by reviews v2 and v3 |

Deploy the sample from the Istio installation directory:

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pods
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze
```

For a local Minikube lab, keep this running in a separate terminal if required by your environment:

```bash
minikube tunnel
```

Open the product page using the address exposed by your gateway setup. In the original Minikube lab, this is commonly `http://localhost/productpage`.

**Why can ratings disappear?** The reviews service has multiple versions. Reviews v1 does not call the ratings service, while v2 and v3 do. When requests reach v1, the page has no star ratings.

## 7. Secure workloads with mTLS

Istio uses `PeerAuthentication` to define the mTLS mode accepted for inbound connections.

| Mode | Behaviour |
|---|---|
| `PERMISSIVE` | Accepts mTLS and plaintext |
| `STRICT` | Accepts only mTLS |
| `DISABLE` | Disables Istio mTLS for the applicable workload (sidecar mode) |

Example: require strict mTLS for workloads in the `default` namespace.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

Apply and inspect:

```bash
kubectl apply -f mtls-strict.yaml
kubectl get peerauthentication -A
```

Allow time for configuration to propagate, then test from a workload that is **not** using the mesh. A plaintext request can fail when the destination requires strict mTLS. A browser request to the Bookinfo gateway can still work because the in-mesh service-to-service connections use workload identities and mTLS.

> **Important:** `PeerAuthentication` governs inbound acceptance. Outbound TLS origination is controlled separately, for example through `DestinationRule`; Istio Auto mTLS can automatically use mTLS for eligible in-mesh traffic when no conflicting TLS policy is configured.

## 8. Manage traffic and roll out a canary

Two resources commonly work together:

- **`DestinationRule`:** defines destination subsets. A subset usually selects Pods by labels such as `version: v1`.
- **`VirtualService`:** routes requests to hosts or subsets and can assign traffic weights.

### Step 1: define the reviews subsets

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
```

### Step 2: send all traffic to v1

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 100
```

### Step 3: split traffic between v1 and v3

Change the route section to:

```yaml
http:
  - route:
      - destination:
          host: reviews
          subset: v1
        weight: 50
      - destination:
          host: reviews
          subset: v3
        weight: 50
```

### Step 4: apply and verify

```bash
kubectl apply -f destination-rule-reviews.yaml
kubectl apply -f virtual-service-reviews.yaml

kubectl get virtualservice reviews -o yaml
kubectl get destinationrule reviews -o yaml
```

When the new version is healthy, shift traffic to 100% v3. The Pods do not need to change for each traffic-weight adjustment; the proxies receive updated routing configuration.

![Canary traffic split](assets/slides/slide-07.png)

A weighted split is a routing target, not a guarantee that every small sample of requests will be exactly proportional. Use metrics and a sufficiently large observation window to make rollout decisions.

## 9. Observability with Kiali

Istio proxies produce telemetry about traffic. Prometheus commonly collects metrics; Kiali uses telemetry to visualise service relationships and health. Add-ons such as Grafana and Jaeger may also be installed for dashboards and tracing.

Install the sample add-ons:

```bash
kubectl apply -f samples/addons
```

Open Kiali:

```bash
istioctl dashboard kiali
```

Choose the `default` namespace and inspect the **Graph** view. Generate application traffic first if the graph is empty.

## 10. Gateway vs Kubernetes Ingress

| Area | Kubernetes Ingress | Istio Gateway + VirtualService |
|---|---|---|
| What it is | Standard Kubernetes API object implemented by an ingress controller | Istio Gateway configures the edge proxy; VirtualService defines routes |
| Routing | Host/path rules; extra features depend on controller | Istio routing features such as weights, retries and timeouts |
| Scope | Primarily cluster edge | Can use a consistent routing model at the edge and between services |

Istio also supports the Kubernetes Gateway API. Check the current Istio documentation for the recommended API and examples for the version you install.

## 11. Troubleshooting

| Symptom | Likely cause | What to check |
|---|---|---|
| Pod shows `1/1`, not `2/2` | Namespace not labelled, injection disabled, or Pod predates the label | `kubectl get ns default --show-labels`; recreate Pods |
| Plaintext request fails after `STRICT` | Expected for a caller outside the mesh | Test from a meshed workload or review the policy scope |
| Traffic split does not behave as expected | Host, subset or label mismatch | `istioctl analyze`; inspect both routing resources |
| Requests to a subset fail | Subset missing or labels do not match | Check `DestinationRule` and Pod labels |
| Browser cannot reach the app | Gateway or local tunnel not configured | Check gateway resources and `minikube tunnel` |
| Updated route appears ignored | Configuration propagation delay or invalid configuration | Wait briefly, then run `istioctl analyze` and inspect resources |
| Kiali graph is empty | Add-ons absent or no traffic generated | Install add-ons and send requests |

Useful inspection commands:

```bash
kubectl get pods -n default
kubectl get virtualservice,destinationrule -o yaml
kubectl get peerauthentication -A
kubectl get gateways
istioctl analyze
```

## 12. Command cheat sheet

| Task | Command |
|---|---|
| Install demo profile | `istioctl install --set profile=demo -y` |
| Enable sidecar injection | `kubectl label namespace default istio-injection=enabled` |
| Analyse Istio config | `istioctl analyze` |
| List mutating webhooks | `kubectl get mutatingwebhookconfiguration` |
| Inspect a VirtualService | `kubectl get virtualservice reviews -o yaml` |
| Inspect mTLS policies | `kubectl get peerauthentication -A` |
| Open Kiali | `istioctl dashboard kiali` |
| Disable automatic injection | `kubectl label namespace default istio-injection-` |
| Uninstall Istio demo installation | `istioctl uninstall -y --purge` |

Clean up the Bookinfo sample before removing Istio:

```bash
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
istioctl uninstall -y --purge
kubectl delete namespace istio-system
```

## 13. Interview questions

**What is a service mesh?**  
A layer that manages service-to-service traffic and adds security, traffic control and observability.

**What is north-south vs east-west traffic?**  
North-south enters or leaves the cluster. East-west flows between services inside the cluster.

**How does Istio implement sidecar-mode traffic management?**  
It injects Envoy proxies into Pods. The proxies intercept workload traffic and enforce configuration distributed by the control plane.

**What is mTLS?**  
Mutual TLS authenticates both peers using certificates and encrypts the connection.

**What is the difference between `PERMISSIVE` and `STRICT`?**  
`PERMISSIVE` accepts mTLS and plaintext; `STRICT` accepts only mTLS.

**How do you configure a canary release?**  
Define subsets in a `DestinationRule`, then use weighted routes in a `VirtualService` and adjust the weights as the rollout progresses.

**What are admission controllers?**  
Kubernetes API-server components that can mutate or validate requests before objects are persisted.

**How does automatic sidecar injection work?**  
A matching Pod-create request is sent by the API server to an Istio mutating admission webhook, which returns a Pod specification containing the proxy.

**What is the difference between `VirtualService` and `DestinationRule`?**  
A `VirtualService` decides where requests go; a `DestinationRule` defines destination subsets and policies.

## 14. Further considerations

- **Resource cost:** sidecars consume CPU and memory, add some latency and increase operational complexity.
- **Telemetry flow:** Envoy proxies generate telemetry. Prometheus commonly scrapes metrics; Kiali visualises mesh topology and health from telemetry sources. `istiod` primarily distributes configuration and manages workload identity/certificates.
- **Webhook scope:** automatic injection depends on the webhook configuration and namespace/workload selection. The namespace injection label is a common opt-in mechanism.
- **Ambient mode:** Istio also has a sidecar-less data plane mode. Ambient uses node-level components such as `ztunnel` for core L4 capabilities, with optional waypoint proxies for L7 processing. Do not assume every sidecar-specific detail applies unchanged to ambient mode.
- **API versions:** prefer stable API versions such as `networking.istio.io/v1` and `security.istio.io/v1` where supported by the installed Istio release. Older examples may use `v1alpha3`.
- **Version discipline:** pin an Istio release that is supported by your Kubernetes version and consult the matching release documentation.

## Repository assets

- `linkedin-istio-carousel.pdf` — 10-page LinkedIn-ready carousel
- `linkedin-istio-carousel.pptx` — PowerPoint deck; includes two animated GIF asset slides
- `assets/gifs/request-flow.gif` — animated request path through Envoy proxies
- `assets/gifs/canary-rollout.gif` — animated traffic shift from v1 to v3
- `assets/slides/` — slide artwork used by the carousel

## References

- [Istio documentation](https://istio.io/latest/docs/)
- [Istio sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)
- [Istio PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- [Istio TLS configuration](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)
- [Kubernetes admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes dynamic admission control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

---

If you find an issue or want to improve an example, open an issue or submit a pull request. Keep examples aligned with the Istio and Kubernetes versions you test.
