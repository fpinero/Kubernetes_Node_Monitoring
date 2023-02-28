* more references about monitoring in civo
https://www.civo.com/learn/categories/prometheus

* Kubernetes Node Monitoring with Prometheus and Grafana
https://www.civo.com/learn/kubernetes-node-monitoring-with-prometheus-and-grafana

* deploy the Prometheus Operator using the following command:<br />
  (de momento no he encontrado la forma de que no falle si especifico un namespace concreto)
```
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml --force-conflicts=true --server-side 
```
* The Prometheus Operator will deploy and manage our instances of Prometheus. We will define our Prometheus deployment declaratively using the following code:
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  resources:
    requests:
      memory: 400Mi
```

```
kubectl apply -f prometheus-deploy.yaml 
```

* our Prometheus deployment can operate freely within our cluster, we need to give it some permissions using a service account and roles.
    * We can define our permissions using the following code:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring

```
````
kubectl apply -f serviceaccount.yaml 
````

* we can view our Prometheus User Interface by exposing our deployment using the Kubectl port-forward command: <br />
  (the pod is the one running in the namespace default without operator word in its name)
```
kubectl port-forward prometheus-prometheus-0 9090:9090
```

* now We can reach our Prometheus UI at the following URL: http://localhost:9090/
---
* Install Node Exporter in Kubernetes
* Node exporter runs on each Node in the Kubernetes cluster, so we can install it as a Deamonset.

* Kubernetes Daemonset deploys our applications in a way that ensures a copy of the application is running on every node in our cluder.

* We can deploy the Node Exporter using the following code:
```
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter
  name: node-exporter
  namespace: prometheus
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
      labels:
        app: node-exporter
    spec:
      containers:
      - args:
        - --web.listen-address=0.0.0.0:9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        image: quay.io/prometheus/node-exporter:v0.18.1
        imagePullPolicy: IfNotPresent
        name: node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: metrics
          protocol: TCP
        resources:
          limits:
            cpu: 200m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 30Mi
        volumeMounts:
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/sys
          name: sys
          readOnly: true
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: proc
      - hostPath:
          path: /sys
          type: ""
        name: sys
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: node-exporter
  name: node-exporter
  namespace: prometheus
spec:
  ports:
  - name: node-exporter
    port: 9100
    protocol: TCP
    targetPort: 9100
  selector:
    app: node-exporter
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: node-exporter
    serviceMonitorSelector: prometheus
  name: node-exporter
  namespace: prometheus
spec:
  endpoints:
  - honorLabels: true
    interval: 30s
    path: /metrics
    targetPort: 9100
  jobLabel: node-exporter
  namespaceSelector:
    matchNames:
    - prometheus
  selector:
    matchLabels:
      app: node-exporter
```

* Next, we will create the resources in our cluster by running the following command:
```
kubectl apply -f nodeexporter.yaml
```
---
* Service Discovery with Prometheus <br />

Service Discovery is Prometheus’ way of finding our desired endpoints to scrape. Although Prometheus and Node Exporter have been installed in our cluster, they have no way of communicating.

The Node Exporter is collecting Metrics from our operating system by our Prometheus server isn't pulling metrics from our Node Exporter.

The Service Monitor object is Prometheus Custom Resource Definition, enabling us to configure our scrape target.

We can declaratively tell Prometheus the applications and namespace we want to collect metrics from, we can also configure scrape frequency, endpoints, and ports.

The following code tells Prometheus to scrape our /metrics endpoint, where our Node Exporter publishes its metrics.

servicemonitor.yaml:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus
  labels:
spec:
  selector:
    matchLabels:
  namespaceSelector:
    any: true
  endpoints:
    - path: /metrics
    - port: http-metrics
```
```
kubectl apply -f servicemonitor.yaml
```
* Now that Prometheus has been configured to scrape our Node Exporter metrics, we can view them in our User Interface http://localhost:9090/

