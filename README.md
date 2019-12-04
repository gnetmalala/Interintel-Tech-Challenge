# Interintel Tech Challenge
## Security configuration setup on the base image (preferably CentOS 8)
### Base Image
```
    FROM scratch
    ADD centos-8-container.tar.xz /

    LABEL org.label-schema.schema-version="1.0" \
        org.label-schema.name="CentOS Base Image" \
        org.label-schema.vendor="CentOS" \
        org.label-schema.license="GPLv2" \
        org.label-schema.build-date="20190927"

    CMD ["/bin/bash"]
```
We use scratch to because it is the most useful image in the context of building base images (such as debian and busybox) or super minimal images (that contain only a single binary and whatever it requires.

### Security configurations
NTP is required for a number of compliance audits and is general good practice.
Install AIDE - Advanced Intrusion Detection Environment
Enable Secure (high quality) Password Policy by Enabling SHA512 instead of using MD5

```
    FROM scratch
    ADD centos-8-container.tar.xz /

    LABEL org.label-schema.schema-version="1.0" \
        org.label-schema.name="CentOS Base Image" \
        org.label-schema.vendor="CentOS" \
        org.label-schema.license="GPLv2" \
        org.label-schema.build-date="20190927"

    CMD ["/bin/bash"]
    RUN yum install ntp ntpdate && chkconfig ntpd on && ntpdate pool.ntp.org && /etc/init.d/ntpd start 
    RUN yum install aide -y && /usr/sbin/aide --init && cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz && /usr/sbin/aide --check
    RUN authconfig --passalgo=sha512 —update
    
```
## kubernetes master nodes
For us to create a new high availability compatible cluster, we must set the following flags in our kube-up script:
1. MULTIZONE=true - so as to prevent removal of master replicas kubelets from zones different than server’s default zone. This is required when we want to run master replicas in different zones, which is recommended.

2. ENABLE_ETCD_QUORUM_READ=true - this is to ensure that reads from all API servers will return most up-to-date data. If true, reads will be directed to leader etcd replica. Setting this value to true is optional: reads will be more reliable but will also be slower.

Optionally, we can specify a GCP zone where the first master replica is to be created. Set the following flag:

3. KUBE_GCE_ZONE=zone - zone where the first master replica will run.

`MULTIZONE=true KUBE_GCE_ZONE=europe-west1-b  ENABLE_ETCD_QUORUM_READS=true ./cluster/kube-up.sh`

After we've set up this cluster, we can replicate the master node, to have more than one master nodes using the following flags on our kub-up script:

1. KUBE_REPLICATE_EXISTING_MASTER=true - to create a replica of an existing master.
2. KUBE_GCE_ZONE=zone - zone where the master replica will run. Must be in the same region as other replicas’ zones.

`KUBE_GCE_ZONE=europe-west1-c KUBE_REPLICATE_EXISTING_MASTER=true ./cluster/kube-up.sh`

### Best practices for replicating masters

1. Place master replicas in different zones. During a zone failure, all masters placed inside the zone will fail. To survive zone failure, also place nodes in multiple zones.
2. Do not use a cluster with two master replicas. Consensus on a two-replica cluster requires both replicas running when changing persistent state.
3. When we add a master replica, cluster state (etcd) is copied to a new instance. If the cluster is large, it may take a long time to duplicate its state. This operation may be sped up by migrating etcd data directory.

## VIP (virtual IP) setup between the two master nodes , using keepalived
keepalived should be considered a complement to, and not a replacement for HAProxy or nginx. The goal is to provide robust HA, such that no downtime is experienced if one or more nodes go offline. To be exact, keepalived ensures that the VIP, which exposes a service-loadbalancer or an Ingress, is always owned by a live node. The DNS record will simply point to this single VIP (ie. sans RR) and the failover will be handled entirely by keepalived.
                                               ___________________
                                              |                   |
                                              | VIP: 10.4.0.50    |
                                        |-----| Host IP: 10.4.0.3 |
                                        |     | Role: Master      |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
Public ----(example.com = 10.4.0.50)----|-----| Host IP: 10.4.0.4 |
                                        |     | Role: Slave       |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
                                        |-----| Host IP: 10.4.0.5 |
                                              | Role: Slave       |
                                              |___________________|

In the above diagram, one node assumes the role of a Master (negotiated via VRRP), and assumes the VIP. example.com points only to the shared VIP 10.4.0.50, instead of the 3 nodes. If 10.4.0.3 is taken offline, the surviving hosts elect a new master to assume the VIP. This model of HA ensures that the VIP can be reached at all times.

#### Configuration
To expose one or more services use the flag services-configmap. The format of the data is: external IP -> namespace/serviceName. Optionally it is possible to specify forwarding method using : after the service name. The valid options are NAT and DR. For instance external IP -> namespace/serviceName:DR. By default, if the method is not specified it will use NAT. If the service name is left blank, only the VIP will be assigned and no routing will be done. This is useful e.g. if you run HAProxy in another pod on the same machines with hostnetwork in order to forward incoming smtp requests via proxy protocol to postfix.

This IP must be routable within the LAN and must be available. By default the IP address of the pods is used to route the traffic. This means that is one pod dies or a new one is created by a scale event the keepalived configuration file will be updated and reloaded.


(Optional) Install the RBAC policies
If you enabled RBAC in your cluster (ie. kube-apiserver runs with the --authorization-mode=RBAC flag), please follow this section so that keepalived can properly query the cluster's API endpoints.

Create a service account so that keepalived can authenticate with kube-apiserver.

kubectl create sa kube-keepalived-vip
Configure the DaemonSet in vip-daemonset.yaml to use the ServiceAccount. Add the serviceAccount to the file as shown:

    spec:
      hostNetwork: true
      serviceAccount: kube-keepalived-vip
      containers:
        - image: k8s.gcr.io/kube-keepalived-vip:0.11


Configure its ClusterRole. keepalived needs to read the pods, nodes, endpoints and services.

    echo 'apiVersion: rbac.authorization.k8s.io/v1alpha1
    kind: ClusterRole
    metadata:
    name: kube-keepalived-vip
    rules:
    - apiGroups: [""]
    resources:
    - pods
    - nodes
    - endpoints
    - services
    - configmaps
    verbs: ["get", "list", "watch"]' | kubectl create -f -
Configure its ClusterRoleBinding. This binds the above ClusterRole to the kube-keepalived-vip ServiceAccount.

    apiVersion: rbac.authorization.k8s.io/v1alpha1
    kind: ClusterRoleBinding
    metadata:
    name: kube-keepalived-vip
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: kube-keepalived-vip
    subjects:
    - kind: ServiceAccount
    name: kube-keepalived-vip
    namespace: default
    Load the keepalived DaemonSet
Next add the required annotation to expose the service using a local IP

    $ echo "apiVersion: v1
    kind: ConfigMap
    metadata:
    name: vip-configmap
    data:
    10.4.0.50: default/echoheaders" | kubectl create -f -
    Now the creation of the daemonset

    $ kubectl create -f vip-daemonset.yaml
    daemonset "kube-keepalived-vip" created
    $ kubectl get daemonset
    NAME                  CONTAINER(S)          IMAGE(S)                         SELECTOR                        NODE-SELECTOR
    kube-keepalived-vip   kube-keepalived-vip   k8s.gcr.io/kube-keepalived-vip:0.11   name in (kube-keepalived-vip)   type=worker


## Worker Nodes
A node is a worker machine in Kubernetes, previously known as a minion. A node may be a VM or physical machine, depending on the cluster. Each node contains the services necessary to run pods and is managed by the master components. The services on a node include the container runtime, kubelet and kube-proxy.

In order for a worker node to join a cluster, it must trust the cluster’s control plane, and the control plane must trust the worker node. This trust is managed via a shared bootstrap token and a certificate authority (CA) key hash. kubeadm handles the exchange between the control plane and the worker node.


## Monitoring
Monitoring is an important component of observability. With all our applications running along-with the various tools, we need to keep an eye on each as well as the underlying frameworks and infrastructure. This helps us ensure and stay in the know of how each component is performing and whether they are achieving the availability and performance expectations that they are supposed to. Monitoring helps us gain insight into cloud-native systems, and combined with alerting it allows DevOps engineers to observe scaled-out applications. 

Prometheus is an open source toolkit for monitoring and alerting under the Cloud-native Computing Foundation (CNCF). Some of the key features that we like about it are that it pulls metric data from services and each service doesn’t need to push metrics to Prometheus itself. Therefore Services need to only expose endpoints where they publish their metrics, and prometheus will call this endpoint and collect the metrics by itself.

Monitors Kubernetes cluster using Prometheus. Shows overall cluster CPU / Memory / Filesystem usage as well as individual pod, containers, systemd services statistics. Uses cAdvisor metrics only.

We need to have running Kubernetes cluster with deployed Prometheus. Prometheus will use metrics provided by cAdvisor via kubelet service (runs on each node of Kubernetes cluster by default) and via kube-apiserver service only.

The Prometheus configuration has to contain following scrape_configs:

        scrape_configs:
        - job_name: kubernetes-nodes-cadvisor
            scrape_interval: 10s
            scrape_timeout: 10s
            scheme: https  # remove if you want to scrape metrics on insecure port
            tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
            kubernetes_sd_configs:
            - role: node
            relabel_configs:
            - action: labelmap
                regex: __meta_kubernetes_node_label_(.+)
            - target_label: __address__
                replacement: kubernetes.default.svc:443
            - source_labels: [__meta_kubernetes_node_name]
                regex: (.+)
                target_label: __metrics_path__
                replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
            metric_relabel_configs:
            - action: replace
                source_labels: [id]
                regex: '^/machine\.slice/machine-rkt\\x2d([^\\]+)\\.+/([^/]+)\.service$'
                target_label: rkt_container_name
                replacement: '${2}-${1}'
            - action: replace
                source_labels: [id]
                regex: '^/system\.slice/(.+)\.service$'
                target_label: systemd_service_name
                replacement: '${1}'

## Centralized log server
Centralized logging is an important component of any production-grade infrastructure, but it is especially critical in a containerized architecture. If you’re using Kubernetes to run your workloads, there’s no easy way to find the correct log “files” on one of the many worker nodes in your cluster. Kubernetes will reschedule your pods between different physical servers or cloud instances. Pod logs can be lost or, if a pod crashes, the logs may also get deleted from disk. Without a centralized logging solution, it is practically impossible to find a particular log file located somewhere on one of the hundreds or thousands of worker nodes.

One of the best solutions is to establish one logging agent per Kubernetes node to collect all logs of all running containers from disk and transmit these to either Elasticsearch, CloudWatch, or both at the same time (a good option available in Kublr with fluentd). When a log collector reads a message, it parses and sorts or filters the results — creating new fields or removing fields — before sending to target storage. You have full control over how your final log data will be presented in Elasticsearch or CloudWatch.

Another important use case for centralized logging is alerting based on detected anomalies such as a sudden drop in traffic volume or an unusual increase in error messages from a particular service in your cluster. Sometimes health checks aren’t enough to determine if the service is running correctly.

Structured logging, is a strategy where applications should output all (or at least most) of their log messages in a structured format like JSON or XML. JSON can be parsed easily, and you can search the logs later based on a particular field value instead of free text. This allows us to create dashboards for BI or analytics teams, and fetch precise aggregated results on-demand for use in periodic reports (for example, number of clients from each country who accessed a service last month or determining the most frequent cause of application crashes to help prioritize).
Another advantage of structured logging versus unstructured or plain-text logs is Elasticsearch performance during querying. When you have many large plain-text messages, the query will take longer to complete than a query by separate field terms, and the results will not be suitable for a dashboard graph or chart. Using structured logging is also critical for ElastAlert, because if one or more free-text queries are run periodically in ElastAlert (especially when you create many alert rules) it will put an unnecessary extra load on the Elasticsearch cluster.