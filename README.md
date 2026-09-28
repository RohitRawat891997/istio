# Istio Practice Labs

This repository is a beginner-friendly set of Istio labs focused on real-world traffic routing, fault injection, and service mesh concepts. Each file is a practice lab built from shell commands and Kubernetes manifests that you can follow step by step.

## What you will learn

- How traffic enters the mesh through a Gateway
- How VirtualService routes requests to services
- How DestinationRule creates subsets and weighted load balancing
- How to match requests using HTTP headers
- How to inject failures and test resilience
- How to verify behavior with kubectl and curl

## Prerequisites

Before you start, make sure you have:

- A running Kubernetes cluster
- Istio installed and working
- kubectl configured to the cluster
- Access to the Istio ingress gateway service
- A Linux shell with curl and vim available

## Repository structure

- `02-gw-vs` — Gateway + VirtualService routing flow
- `03-dr-1` — DestinationRule with weighted traffic split
- `03-dr-2` — Advanced header-based routing and canary release demos
- `04-chaos` — Fault injection, retries, and chaos exercises

## Recommended practice flow

1. Start with `02-gw-vs` to understand basic ingress routing.
2. Continue with `03-dr-1` to learn weighted traffic splitting.
3. Practice `03-dr-2` for version routing and HTTP header matching.
4. Finish with `04-chaos` to test failure scenarios and service resilience.

## Quick tips

- Use `kubectl get gw,vs,dr` often to inspect Istio resources.
- Use `kubectl get svc -n istio-system` to find the ingress gateway IP.
- Add hostnames to `/etc/hosts` when testing via browser or curl.
- Read the output carefully: successful traffic, HTTP 500, and response distribution are all part of the learning.

## Important note

These labs were originally captured as command logs. They are intentionally practical and realistic. This README makes the flow easier to understand for learners who want to follow along without prior experience.
