### Setting up a monitoring stack for Bluerise app (NodeJs)

If you have an aplicaton that is exposed to traffic, youll want to monitor its stability and health. 
This documentation highlights the use of prometheus-stack (Prometheus, Grafana and Alertmanager) for mmonitoring and observability of an app and/or infrastructure.

I will *simplify* the use of a monitoring stack using single node kubernetes cluster.

### Requirements
* The NodeJs application + prom-client library installed +  metrics endpoint exposed
* Jenkins for CI/CD
* KIND  for Kubernetes cluster
* Prometheus-stack f
        * Prometheus    - Monitoring
        * Grafana           - Visaualization
        * Alertmanager  - Handles Alerts from the       Prometheus server




### 1. NodeJs Application Setup
 *Bluerise weather app will be modified with a **prom-client** dependency to ensure a **/metric** endoint is made available for **Prometheus** to scrape.*

source code below -s
```
require('dotenv').config();
const express = require('express');
const axios = require('axios');
const path = require('path');
const fs = require('fs');
const client = require('prom-client');  // 📈 Add prom-client

const app = express();
const API_KEY = process.env.WEATHER_API_KEY;
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));

// 📊 Prometheus metrics setup
const register = new client.Registry();
client.collectDefaultMetrics({ register }); // collects CPU, memory, etc.

const weatherCheckCounter = new client.Counter({
  name: 'weather_checks_total',
  help: 'Total number of weather data fetches',
});
register.registerMetric(weatherCheckCounter);

// Routes
app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.post('/weather', async (req, res) => {
  const city = req.body.city;
  const url = `http://api.weatherapi.com/v1/current.json?key=${API_KEY}&q=${encodeURIComponent(city)}`;

  try {
    const response = await axios.get(url);
    const weather = response.data;

    // 👇 Increment weather check counter
    weatherCheckCounter.inc();

    let responseHtml = fs.readFileSync(path.join(__dirname, 'public', 'response.html'), 'utf8');
    responseHtml = responseHtml.replace('{{city}}', weather.location.name);
    responseHtml = responseHtml.replace('{{country}}', weather.location.country);
    responseHtml = responseHtml.replace('{{temp}}', weather.current.temp_c);
    responseHtml = responseHtml.replace('{{condition}}', weather.current.condition.text);
    responseHtml = responseHtml.replace('{{humidity}}', weather.current.humidity);
    responseHtml = responseHtml.replace('{{wind}}', weather.current.wind_kph);

    res.send(responseHtml);
  } catch (err) {
    res.send(`
      <p>Error: Could not fetch weather data. Please check the city name.</p>
      <a href="/">Try again</a>
    `);
  }
});

// 📊 Metrics endpoint for Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(PORT, () => {
  console.log(`🌐 Server is running on http://localhost:${PORT}`);
});
``` 

*If you have your own custom NodeJs code, ensure to specify the prom-client and Prometheus bits in it* E.g 

``` Install prom-client ```

```const client = require('prom-client');  // 📈 Add prom-client ```

```// 📊 Prometheus metrics setup
const register = new client.Registry();
client.collectDefaultMetrics({ register }); // collects CPU, memory, etc.

const weatherCheckCounter = new client.Counter({
  name: 'weather_checks_total',
  help: 'Total number of weather data fetches',
});
register.registerMetric(weatherCheckCounter);
```

```
// 📊 Metrics endpoint for Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```
*Ensure/update **package.json** file lists **prom-client** as one of the dependencies, this will be helpful when using Dockerfile for the image build.*

Run the application to check it runs successfully, ```node index.js``` 

**Containerise the Application** - (Build an Image)
* Create a dockerfile 

* Specify  commands to build the application as an image --

``` 
# Use the official Node.js image as the base image
FROM node:18

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy the package.json and package-lock.json (if available)
COPY package*.json ./

# Install dependencies declared in package.json
RUN npm install 

# Copy the rest of the application code
COPY . .

# Expose the port the app will run on (for example, port 3000)
EXPOSE 3000

# Command to run the app
CMD ["npm", "start"]
```

We will use this dockerfile in an automated way and not manually by just running **docker build** command. For the automation, we **Jenkins** as a **CI/CD** tool. 
Jenkins is already running on our local machine and will be used to pull the source code from our repo, build the image, tag it, and push it to the registry, all in one command. The configuration for this task will be specified in the Jenkinsfile. A guide to setup a Jenkins pipeline, click [here]([Jenkins](https://www.jenkins.io/doc/book/pipeline/))

The Jenkinsfile
Create a jenkins with configurations to check out the repo, build the image using the dockerfile, push image to the registry and then logout.

JenkinsFile  -
```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'johnstx/bluerise'
        IMAGE_TAG = 'v1.3'
        REGISTRY_CREDENTIALS_ID = 'dockerhub-login'  // Jenkins credentials ID
    }

    stages {


        stage('Clean Workspace') {
          steps {
                    cleanWs() // Cleans the workspace before running the pipeline
          }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}

```
Now the image of the app has been created and deployed in the dockerhub registry, ready to be pulled and utilised by multiple users/teams.
The image will be used to deploy a microservices application running in a Cluster. 


##  Kubernetes Cluster set up
☸️ **Kubernetes Setup (KIND)**

🧪 **Create Cluster**

```
 kind create cluster --name bluerise
``` 

1. Deploy the bluerise deployments and service in the default namesapace, you can create a unique namespace for this aswell.

```kubectl apply -f bluerise-deployment.yaml```
```kubectl apply -f bluerise-service.yaml```


### 📊 Prometheus + Grafana Monitoring Stack

**Add Helm Repo & Install Stack** (Using HELM, install prometheus-stack helm chart. )
*Create a monitoring namepace, install a release of the  prometheus-stack in this namespace called monitors.*
*Deploy ServiceMonitor - to scrape the /metrics endpoint of the application*

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitors prometheus-community/kube-prometheus-stack -n monitoring
```

**Create a ServiceMonitor to explicitly tell Prometheus to scrape the service.**

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bluerise-monitor
  namespace: monitoring
  labels:
    release: monitors  # Must match the Helm release label
spec:
  selector:
    matchLabels:
      app: bluerise
  namespaceSelector:
    matchNames:
      - default
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```


##  Step 3: Access Prometheus and Grafana Dashboards
📊 **Port-forward Prometheus:**

```kubectl port-forward svc/monitors-kube-prometheus-s-prometheus -n monitoring 9090:80```
Open: **http://localhost:9090/targets**

Confirm that bluerise appears in the "targets" list and is marked UP.


 **Port-forward Grafana:**
kubectl port-forward svc/monitors-grafana -n monitoring 3000:80


Now, Go ahead and create your dashboard to suit the metric you want to monitor

Also read & follow @ https://medium.com/@inyiri.io




## SomeBugs & Fixes

**1.Prometheus failed to show any metrics from the app**
This is because the endpoint in the NodeJs **index.js** file was not specified. 
**Fix**
1. Install prom-client
2. Modify the index.js file to specify the client, prometheus setup and the metrics endpoint for prometheus.

Test the app and check the /metrics endpoint in your browser. You should find a records on this page.
## success. OK
 Then push to your repo and pipeline.


 http://localhost:9090/targets run this to see the targets prometheus scrapes


**2. Servicemonitor unable to find the deployment/service**

 **Fix**
*kubectl describe* on servicemonitor revealed the it was not exactly scraping any source.
Ensure the  *port* section in the *bluerise-service.yaml* file has the *name: http* WHILE 
the *bluerise-servicemonitor.yaml* should have the  *port: http* specified in the *endoints* section.

**3. Empty /metrics endpoint**
If the *http://localhost:3000* is empty

**Fix**
Check that **package.json** fine lists prom-client as a dependency. 
**NB** The node_modules directory should also list *prom-client* by now, after the install. 