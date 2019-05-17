# Deploy CockroachDB Statefulset to Kubernetes Cluster

## Specs

For our experiments, we need to ensure that our CockroachDB pods run with the following specs:
- 14 vCPU and 30 GB RAM resource request
- 16 vCPU and 60 GB RAM resource limit
- 500 GB persistent volume claim

## Steps

The following steps will deploy CockroachDB statefulset on our Kubernetes cluster with the required specs.

1. Create statefulset

    Create a file named `cockroachdb-statefulset.yaml` with content as follow:

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: cockroachdb-public
      labels:
        app: cockroachdb
    spec:
      ports:
      - port: 26257
        targetPort: 26257
        name: grpc
      - port: 8080
        targetPort: 8080
        name: http
      selector:
        app: cockroachdb
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: cockroachdb
      labels:
        app: cockroachdb
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "_status/vars"
        prometheus.io/port: "8080"
    spec:
      ports:
      - port: 26257
        targetPort: 26257
        name: grpc
      - port: 8080
        targetPort: 8080
        name: http
      publishNotReadyAddresses: true
      clusterIP: None
      selector:
        app: cockroachdb
    ---
    apiVersion: policy/v1beta1
    kind: PodDisruptionBudget
    metadata:
      name: cockroachdb-budget
      labels:
        app: cockroachdb
    spec:
      selector:
        matchLabels:
          app: cockroachdb
      maxUnavailable: 1
    ---
    apiVersion: apps/v1beta1
    kind: StatefulSet
    metadata:
      name: cockroachdb
    spec:
      serviceName: "cockroachdb"
      replicas: 3
      template:
        metadata:
          labels:
            app: cockroachdb
        spec:
          hostNetwork: true
          dnsPolicy: ClusterFirstWithHostNet
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - cockroachdb
                  topologyKey: kubernetes.io/hostname
          containers:
          - name: cockroachdb
            image: cockroachdb/cockroach:v19.1.0
            imagePullPolicy: IfNotPresent
            resources:
              requests:
                cpu: "14"
                memory: "30Gi"
              limits:
                cpu: "16"
                memory: "60Gi"
            ports:
            - containerPort: 26257
              name: grpc
            - containerPort: 8080
              name: http
            livenessProbe:
              httpGet:
                path: "/health"
                port: http
              initialDelaySeconds: 30
              periodSeconds: 5
            readinessProbe:
              httpGet:
                path: "/health?ready=1"
                port: http
              initialDelaySeconds: 10
              periodSeconds: 5
              failureThreshold: 2
            volumeMounts:
            - name: datadir
              mountPath: /cockroach/cockroach-data
            env:
            - name: COCKROACH_CHANNEL
              value: kubernetes-insecure
            command:
              - "/bin/bash"
              - "-ecx"
              - "exec /cockroach/cockroach start --logtostderr --insecure --advertise-host $(hostname -f) --http-addr 0.0.0.0 --join cockroachdb-0.cockroachdb,cockroachdb-1.cockroachdb,cockroachdb-2.cockroachdb --cache 25% --max-sql-memory 25%"
          terminationGracePeriodSeconds: 60
          volumes:
          - name: "datadir"
            hostPath:
              path: /mnt/disks/ssd0
          nodeSelector:
            cloud.google.com/gke-local-ssd: "true"
      podManagementPolicy: Parallel
      updateStrategy:
        type: RollingUpdate
      volumeClaimTemplates:
      - metadata:
          name: datadir
        spec:
          accessModes:
          - "ReadWriteOnce"
          storageClassName: local-ssd
          resources:
            requests:
              storage: 500Gi
    ```

    Apply it:

    ```
    kubectl apply -f cockroachdb-statefulset.yaml
    ```

2. Init cluster

    Create a file named `cluster-init.yaml` with content as follow:

    ```
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: cluster-init
      labels:
        app: cockroachdb
    spec:
      template:
        spec:
          containers:
          - name: cluster-init
            image: cockroachdb/cockroach:v19.1.0
            imagePullPolicy: IfNotPresent
            command:
              - "/cockroach/cockroach"
              - "init"
              - "--insecure"
              - "--host=cockroachdb-0.cockroachdb"
          restartPolicy: OnFailure
    ```

    Apply it:

    ```
    kubectl apply -f cluster-init.yaml
    ```

3. Setup database

    Access CockroachDB client with the following command:

    ```
    kubectl run cockroachdb -it --image=cockroachdb/cockroach --rm --restart=Never -- sql --insecure --host=cockroachdb-public
    ```

    Once inside, create a test database:

    ```
    create database test;
    \q
    ```

4. Setup load balancer

    Expose `cockroachdb-public` service:

    ```
    kubectl expose service cockroachdb-public --port=26257 --target-port=26257 --name=cp --type=LoadBalancer
    ```

    To verify that it is working, run `kubectl get services`, the result should look something like this:

    ```
    NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)              AGE
    cockroachdb          ClusterIP      None            <none>          26257/TCP,8080/TCP   10m
    cockroachdb-public   ClusterIP      10.11.251.158   <none>          26257/TCP,8080/TCP   10m
    cp                   LoadBalancer   10.11.242.253   35.229.206.21   26257:32471/TCP      2m8s
    kubernetes           ClusterIP      10.11.240.1     <none>          443/TCP              49m
    ```

    You will need the external IP to run `go-ycsb`.