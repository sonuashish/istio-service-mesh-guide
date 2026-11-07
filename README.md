# Istio Service Mesh Field Guide

A practical Kubernetes/Istio learning repository covering Envoy sidecars, admission webhooks, mTLS, canary traffic management, Bookinfo, Kiali and troubleshooting.

## Start here

- Read the [complete guide](docs/istio-service-mesh-guide.md).
- View the [LinkedIn carousel PDF](linkedin-istio-carousel.pdf).
- Open the [PowerPoint source deck](linkedin-istio-carousel.pptx).
- Explore the animated GIFs in `assets/gifs/`.

## Contents

```text
.
├── README.md
├── docs/
│   └── istio-service-mesh-guide.md
├── linkedin-istio-carousel.pdf
├── linkedin-istio-carousel.pptx
└── assets/
    ├── gifs/
    └── slides/
```

## Lab prerequisites

- Kubernetes cluster (Minikube, kind, or another supported cluster)
- `kubectl`
- An Istio release compatible with your Kubernetes version
- Basic Kubernetes concepts: Pods, Services, Deployments and namespaces

## Important

The commands and manifests are learning examples. Verify API compatibility and release support against the official Istio documentation before using them in a real cluster. The demo profile is not a production installation profile.
