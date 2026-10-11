Kubernetes Scheduling Lab

A hands-on Kubernetes lab for learning pod scheduling and placement concepts.

Topics Covered

Taints and tolerations

Node affinity and anti-affinity

Pod affinity and anti-affinity

Deployment and ReplicaSet management

Pod scheduling and troubleshooting

Prerequisites

Docker Desktop

Kubernetes

kubectl

kind (Kubernetes IN Docker)

Verify the Cluster

kubectl get nodes -o wide
kubectl get pods -A -o wide

Apply a Manifest

kubectl apply -f payment.yml
kubectl get pods -l app=payment -o wide

Troubleshooting

Describe a pod to inspect scheduling events:

kubectl describe pod <pod-name>

View Deployment and ReplicaSet status:

kubectl get deployments,replicasets

Purpose

This repository documents practical Kubernetes scheduling exercises and troubleshooting scenarios.