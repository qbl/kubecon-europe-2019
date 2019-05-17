# Setup Kubernetes Cluster on GKE

## Specs

To run the experiments properly, we need a Kubernetes cluster with 3 nodes with each node has the following specs:
- n1-standard-16 machine
- 1000 GB local SSD

We also need to make sure that our cluster has the required storage class and persistent volume to utilize local SSD in the cluster.

## Steps

The following steps will spin a new Kubernetes cluster on GKE with the required specs.

1. Create cluster
    
    Create a Kubernetes cluster with 3 nodes:
    ```
    gcloud container clusters create benchmark-sandbox --machine-type=n1-standard-16 --num-nodes 3 --local-ssd-count 3 --disk-size=1000
    ```

    Get the names of all nodes, we will need this for persistent volume:
    ```
    kubectl get nodes
    ```

    The result should look like this:

    ```
    NAME                                               STATUS   ROLES    AGE   VERSION
    gke-benchmark-sandbox-default-pool-99da7494-1hm2   Ready    <none>   16m   v1.12.7-gke.10
    gke-benchmark-sandbox-default-pool-99da7494-4zt0   Ready    <none>   16m   v1.12.7-gke.10
    gke-benchmark-sandbox-default-pool-99da7494-q2s9   Ready    <none>   16m   v1.12.7-gke.10
    ```

2. Create a storage class
    
    Create a file named `local-storageclass.yaml` with content as follow:

    ```
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-ssd
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ```

    Apply it:

    ```
    kubectl apply -f local-storageclass.yaml
    ```

3. Create persistent volumes:

    Create a file named `local-persistentvolume.yaml` with content as follow:

    ```
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: volume-0
    spec:
      capacity:
        storage: 500Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-ssd
      local:
        path: /mnt/disks/ssd0
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: "kubernetes.io/hostname"
              operator: "In"
              values: ["gke-benchmark-sandbox-default-pool-99da7494-1hm2"]

    ---

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: volume-1
    spec:
      capacity:
        storage: 500Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-ssd
      local:
        path: /mnt/disks/ssd0
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: "kubernetes.io/hostname"
              operator: "In"
              values: ["gke-benchmark-sandbox-default-pool-99da7494-4zt0"]

    ---

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: volume-2
    spec:
      capacity:
        storage: 500Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-ssd
      local:
        path: /mnt/disks/ssd0
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: "kubernetes.io/hostname"
              operator: "In"
              values: ["gke-benchmark-sandbox-default-pool-99da7494-q2s9"]
    ```

    Apply it:

    ```
    kubectl apply -f local-persistentvolume.yaml
    ```