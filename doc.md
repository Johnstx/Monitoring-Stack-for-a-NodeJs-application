kind load docker-image bluerise:latest --name my-cluster

How I deployed my nodejs weather App into Kubernetes

This setup replicates a deployment of an enterprise microservice. 
I used (Kubernetes IN Docker) kubernetes cluster, easy for low payload launches into kubernetes. Therefore ensure you have docker desktop on your engine when you want to use KIND
The application to be lauched here is a custom application previously built as a microservices app.

Deploying an Application into A kubernetes cluster in easy steps.

1. Install KIND
 Click [Kind](https://kind.sigs.k8s.io/#:~:text=You%20can%20install%20kind%20with,also%20need%20to%20install%20docker.) to install.

2. Create a Kubernetes CLUSTER
Run

```
 kind create cluster --name bluerise
```
![alt text](<images/1 kind create cluster.jpg>)

Interact with the new cluster a bit

```
kubectl cluster-info --context kind-bluerise
```

3. LOAD image of the application/s for your workflow

```
kind load docker-image johnstx/bluerise:v1.3
``` 

4. DEPLOY the application manifest files. (Deployments, Services etc)

bluerise-deployment.yaml - 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bluerise-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bluerise
  template:
    metadata:
      labels:
        app: bluerise
    spec:
      containers:
      - name: bluerise-weather-app
        image: johnstx/bluerise:v1
        ports:
        - containerPort: 3000

```

```
kubectl apply -f bluerise-deployment.yaml
```

### Manifest file for the service 

bluerise-service.yaml 

```
apiVersion: v1
kind: Service
metadata:
  name: bluerise-service
spec:
  type: NodePort
  selector:
    app: bluerise
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 30081
```

```
kubectl apply -f bluerise-service.yaml
``` 


Check out the resources for more info using *kubectl get*

![alt text](<images/4 get pods.jpg>)

![alt text](<images/5 get svc.jpg>)


### Access the application through the browser

Port-forward the service to enable http access.

```
kubectl port-forward svc/bluerise 8080:80
``` 

![alt text](<images/6 port-forward.jpg>)


Accessing the app on then browser

![alt text](<images/7 browser.jpg>)




helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install monitor prometheus-community/kube-prometheus-stack -n monitoring


NAME: monitors
LAST DEPLOYED: Sun Jun 15 00:18:50 2025
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitors"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets monitors-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitors" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator. 



BUGS
After installing the helm release, only the node exporter pod was running, the other thre pods were stuck on "containerCreating for a while", then the metrics container came on leaving the operator and grafana pods -- All came on but one pod for the monitors-grafana.

### Create a ServiceMonitor to explicitly tell Prometheus to scrape the service.

###  Step 3: Access Prometheus and Grafana Dashboards
📊 Port-forward Prometheus:
kubectl port-forward svc/monitors-kube-prometheus-s-prometheus -n monitoring 9090
Open: http://localhost:9090/targets

Confirm that bluerise appears in the "targets" list and is marked UP.


kubectl port-forward svc/monitors-grafana -n monitoring 3000:80



### Prometheus failed to show any metrics from the app.?
This is because he endpoint in the nodejs application config was not specified. So I have to revisit the bluerise application, and the pipeline for docker build. This is where you'd appreciate CI/CD. 
So we are going to modify the index.js file to include the prom-client and /metrics endpoint for prometheus to scrap info from it. Then using jenkins to build the image. 
1. install prom-client
2. modify the index.js file to specify the client, prometheus setup and the metrics endpoint for prometheus.

### New Bug - 
/metrics endopoint EMPTY!!

### Bug fix
Check that **package.json** fine lists prom-client as a dependency. 
**NB** The node_modules directory should also list *prom-client* by now, after the install. 

Test the app and check the /metrics endpoint in your browser. You should find a records on this page.
## success. OK
 Then push to your repo and pipeline.


 http://localhost:9090/targets run this to see the targets prometheus scrapes

 ### Bug
 Servicemonitor for my app was not listed above, but could be seen when *k get servicemonitors* 
 ### Bugfix
 *describe* on app's servicemonitor revealed the it was not exactly scraping any source, as noticed from the *port* section of the code which should have the *name* of the port as *http* (it is supposeed to scraping the app service) of the app's service directory. HOwever, the service doesnt have a *name* specified as *http*, so it leaves an empty targt for the servicemonitor. Solution is to modify the *port* field of the service by adding **name:http** to it. 