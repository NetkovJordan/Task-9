# DevOps deployment exercise -  Deploy application and monitor with Prometheus and Grafana using Kubernetes

## Introduction

In this exercise we will focus on deploying a sample application and setting up monitoring with Prometheus and Grafana using Kubernetes. This task will guide you through the deployment of a containerized application into a Kubernetes cluster, and then integrate Prometheus for collecting and storing metrics, alongside Grafana for visualizing those metrics. By the end of this exercise, you will have a fully operational monitoring stack that provides real-time insights into the performance and health of your application, allowing for more efficient management and troubleshooting.
The application in question is from public Dockerhub registry: `netkovjordan/shark-app:2.0` The application is shark glorifying!
It also has an additional service which exposes metrics to prometheus on `/metrics`, you can test this out in the browser after deployment.
We have 2 phases of the exercise, in which we first deploy using commands, and in the second we will leverage the kubernetes-dashboard UI for ease of use.

### Kubernetes CLI deployment (We use docker-desktop as cluster context)

Here we use helm to deploy ingress controller, kube state metrics which will expose cluster metrics to prometheus and later we will deploy
Kubernetes dashboard with helm. The ingress nginx controller implements the ingress resources defined from the manifests and exposes the services externally. Kube state metrics is a service by prometheus community, which exposes cluster metrics and Prometheus scrapes these metrics. The last
helm repository used is Kubernetes dashboard which is an interactive UI for better vision and monitoring on Kubernetes cluster.

1. Make sure to have helm installed (can be done via Chocolatey on Windows)

2. Run the following commands: 


```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx-controller --create-namespace --namespace ingress ingress-nginx/ingress-nginx -f ingress.yaml
helm install kube-state-metrics prometheus-community/kube-state-metrics -n kube-system
```

Verify the pods are running:
```
kubectl get pods -n ingress -w
kubectl get pods -n kube-system -w
```


### Deployment manifest explanation

This Kubernetes configuration sets up a simple application deployment. First, it creates a `Namespace` called `app-namespace` to organize resources. Within this namespace, a `Deployment` named `shark-app` is created with 3 replicas, each running a container from the `netkovjordan/shark-app:2.0` image. This container listens on port 8080 and has memory and CPU limits set. A `Service` named `shark-service` is then defined to expose the application, routing traffic from port 82 to the container's port. Finally, an `Ingress` resource named `app-ingress` is configured to manage external access to the application, directing requests from shark.applications.com to the shark-service.


3. Run `kubectl apply -f .\deployment.yaml`

`kubectl get pods -w -n app-namespace`

Check if app is available in browser on shark.applications.com

### Prometheus manifest explanation

This manifest sets up a Prometheus monitoring system in Kubernetes. It includes a `ConfigMap` that defines the Prometheus configuration, specifying global settings and scraping intervals for two services: `shark-service` and `kube-state-metrics`. The `Deployment` creates a Prometheus pod using this configuration, exposing Prometheus on port 9090. The `Service` makes Prometheus accessible within the cluster on port 3003. Finally, the `Ingress` defines an external URL (prometheus.example.com) to access Prometheus, routing traffic to the Prometheus service through NGINX.


4. Run `kubectl apply -f .\prometheus.yaml`

`kubectl get pods -n app-namespace`

Check in the browser to see if prometheus.example.com is available.

### Grafana manifest explanation

This Kubernetes configuration sets up Grafana with secure credentials, persistent storage, and accessible endpoints. First, two Secret resources are created in the `app-namespace` namespace: `grafana-username` and `grafana-password`, which store the encoded credentials for Grafana's admin user. Next, a `PersistentVolumeClaim` named `grafana-pvc` is defined to request 1Gi of storage for Grafana's data, ensuring data persists even if the pod is recreated. Then, a `Deployment` named `grafana-deployment` is set up, deploying the Grafana application using the `grafana/grafana` image. The deployment uses the secrets for the admin username and password, sets resource requests and limits, exposes port 3000, and mounts the persistent volume for storage. A `Service` named `grafana-service` is created to expose Grafana on port 3000. Finally, an `Ingress` named `grafana-ingress` is defined to manage external access to Grafana, directing traffic from grafana.example.com to the grafana-service on port 3000.

5. Run `kubectl apply -f .\grafana.yaml`

`kubectl get pods -w -n app-namespace`

Try to access grafana.example.com in the browser

You should see this:

```
kubectl get pods -n app-namespace
NAME                                     READY   STATUS    RESTARTS   AGE
grafana-deployment-6c85c76fc6-snzd4      1/1     Running   0          9m45s
prometheus-deployment-64d8998467-lqj7j   1/1     Running   0          22m
shark-app-649c6dd9b-g457x                1/1     Running   0          45s
shark-app-649c6dd9b-jf9kr                1/1     Running   0          45s
shark-app-649c6dd9b-lx9zv                1/1     Running   0          45s
```

6. Now log in to Grafana, add Prometheus as Datasource, create dashboards for shark app (using Node.js metrics). Also create dashboards using kubernetes metrics. Leverage metrics exporter to see which metrics correspond to which visualizations.

7. Now in order to clean up (without removing kube state metrics and ingress nginx) we can just remove the app-namespace:

Run `kubectl delete ns app-namespace`

Confirm if the namespace is deleted:

`kubectl get ns`

### Deploying via Kubernetes Dashboard

1. Install the dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```
2. Make sure we have `cluster-admin` as `Clusterrole`

Run `kubectl get clusterrole`

Should see `cluster-admin` in Cluster roles.

This role is created by default by Kubernetes, and as you can imagine it grants admin privileges to a user.

3. Change directory to ./kubernetes-dashboard

`cd kubernetes-dashboard`

4. Apply the service account manifest

`kubectl apply -f .\sa.yaml` 

This manifest creates a service account named admin-user

5. Apply the clusterrolebinding manifest:

`kubectl apply -f .\crb.yaml`

What this does is it references the existing `ClusterRole` `cluster-admin` (does reading all of the permissions granted to this role)
and assigns it to a user, group or in our case a Service account.

Now why do we create these? Well, in order to grant the Kubernetes dashboard access to all cluster information and resources, which will be also interacted with in the browser, we need to tell the kubernetes dashboard that we are an authorized user with admin permissions to the cluster in order for all information to be available.

6. Next we expose the service:

`kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443`

7. We create an access token for the service account in order to authorize with the dashboard:

`kubectl create token admin-user -n kubernetes-dashboard`

We copy the token and login.

Now our next job is to deploy each of these manifests in the Dashboard UI, monitor the pods, view configmaps and services, inspect the logs of the pods. Try deleting pods and see what happens in the UI. Play with the dashboard, maybe deploy other services on it.

### Cleaning up

1. In order to remove everything, we can first delete the app-namespace again:

`kubectl delete ns app-namespace`

2. Now to remove the helm installations:
```
helm uninstall kubernetes-dashboard -n kubernetes-dashboard
helm uninstall kube-state-metrics -n kube-system
helm uninstall kubernetes-dashboard -n kubernetes-dashboard
```

3. Finally, remove the remaining namespaces:

`kubectl delete ns ingress kubernetes-dashboard`

If everything is cleaned up, we should only have this:
```
kubectl get ns
NAME              STATUS   AGE
default           Active   19h
kube-node-lease   Active   19h
kube-public       Active   19h
kube-system       Active   19h
```



## Conclusion

This is just a short task, in which we utilize more services, all sitting behind NGINX. We show how a normal website or even a more complex web application could work when deployed on a Kubernetes cluster. We show how Prometheus and Grafana can be used to monitor the health and metrics of an application, in which Prometheus scrapes metrics and sends them to Grafana which visializes them. And then in the second phase we see how we can deploy kubernetes-dashboard, get a mini intro to RBAC (role based access control), and we understand how easier it is to deploy via the dashboard and how more interractive it is and how much more ease of use it offers.
