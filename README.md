# KubeCon + CloudNativeCon Europe 2019

## Description

This repository contains steps to reproduce our experiments on Benchmarking Cloud Native Database Running on Kubernetes.

## Steps

To reproduce experiments, do the following steps:

1. [Setup a Kubernetes cluster on GKE](setup-kubernetes-cluster.md)
2. Install and compile `go-ycsb`
3. Deploy a database to the cluster
4. Load and run each YCSB workload