# KubeCon + CloudNativeCon Europe 2019

## Description

This repository contains steps to reproduce our experiments on Benchmarking Cloud Native Database Running on Kubernetes.

## Steps

To reproduce experiments, do the following steps:

1. [Setup a Kubernetes cluster on GKE](setup-kubernetes-cluster.md)
2. Install and compile `go-ycsb`

    We are going to use `go-ycsb` to run our benchmarking tests. To get install and compile `go-ycsb`, run the following commands:

    ```
    git clone https://github.com/pingcap/go-ycsb.git $GOPATH/src/github.com/pingcap/go-ycsb
    cd $GOPATH/src/github.com/pingcap/go-ycsb
    make
    ```

    To verify that it is working, run:

    ```
    ./bin/go-ycsb
    ```

3. Deploy a database to the cluster

    Choose one of the following instructions to setup a database cluster:

    - [CokcroachDB](deploy-cockroachdb.md)

4. Load and run each YCSB workload