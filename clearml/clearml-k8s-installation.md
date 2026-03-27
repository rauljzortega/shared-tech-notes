# ClearML with Kubernetes

*Official documentation reference: https://github.com/clearml/clearml-helm-charts/tree/main/charts/clearml*  

## 1. Prerequisites

- Install a [Kubernetes cluster](../kubernetes/kubernetes-installation.md)   
- Install [Helm](../kubernetes/helm/helm.md)  
- Install an [Ingress Controller](../kubernetes/helm/ingresscontroller.md)
- Install a [Storage Class](../kubernetes/helm/storageclass.md)

## 2. ClearML Server installation 

Once you have your Kubernetes cluster installed with Helm, follow these steps:  

### 2.1. Add the Helm repository

Add the ClearML chart to your local Helm repository:  
```bash
helm repo add clearml https://clearml.github.io/clearml-helm-charts
```  

### 2.2. Configure your deployment

Clone the ClearML chart repository and edit the `values-production.yaml` file to configure your domain names:  
```bash
git clone https://github.com/clearml/clearml-helm-charts.git
vim clearml-helm-charts/charts/clearml/values-production.yaml
```  
Inside this file you have to edit these lines:  
```yaml
  hostName: "api.<your domain name>"
  hostName: "files.<your domain name>"
  hostName: "app.<your domain name>"
```  

In order to set these values, you can use either a wildcard DNS service or a custom domain name.  
Examples:
- **Using an IP** (Easiest for testing):  
  *Note: use a worker node's IP*  
  ```yaml
  hostName: "api.clearml.172-16-10-11.nip.io"
  hostName: "files.clearml.172-16-10-11.nip.io"
  hostName: "app.clearml.172-16-10-11.nip.io"
  ```  

- **Using a custom domain name** (Production environment):  
  ```yaml
  hostName: "api.clearml.doitnowgroup.local"
  hostName: "files.clearml.doitnowgroup.local"
  hostName: "app.clearml.doitnowgroup.local"
  ```  

Then, you must resolve these names on the computer you will use to access the web UI. Add a line such as this to your local `/etc/hosts` file:  
```
172.16.10.11    api.clearml.doitnowgroup.local files.clearml.doitnowgroup.local app.clearml.doitnowgroup.local
```  

*NOTE: Elasticsearch default deployment is set to 3 replicas. If you are using a small cluster, you should adapt the number of replicas and RAM in your `values-production.yaml` file.*  

### 2.3. Deploy ClearML

Install the ClearML app using your config:  
```bash
helm install clearml clearml/clearml -f clearml-helm-charts/charts/clearml/values-production.yaml
```  


Depending on your Kubernetes version, you might find an error like this:  
```
[rjuarez@controlplane clearml-helm-charts]$ helm install clearml clearml/clearml -f clearml-helm-charts/charts/clearml/values-production.yaml
Error: INSTALLATION FAILED: failed to create typed patch object (default/clearml-mongodb; apps/v1, Kind=StatefulSet): .spec.updateStrategy.rollingUpdate.maxSurge: field not declared in schema
```  
If this happens, follow these steps:
- Edit your file `values-production.yaml`.
- Find the mongodb block and replace it with this block:
  ```yaml
  mongodb:
    enabled: true
    architecture: replicaset
    replicaCount: 2
    updateStrategy:
      type: OnDelete
      rollingUpdate: null
    arbiter:
      enabled: false
    pdb:
      create: true
    podAntiAffinityPreset: soft
  ```  
- Uninstall the failed ClearML release and delete all related data:  
  ```bash
  helm uninstall clearml
  kubectl delete statefulset clearml-mongodb --ignore-not-found
  kubectl delete all -l app.kubernetes.io/instance=clearml
  ```
- Install ClearML again:
  ```bash
  helm install clearml clearml/clearml -f clearml-helm-charts/charts/clearml/values-production.yaml
  ```  
*Note: if you make changes in your config file you can update the running app by executing:*  
```bash
helm upgrade clearml clearml/clearml -f clearml-helm-charts/charts/clearml/values-production.yaml
```

### 2.4. Verifying the installation

Check the status of your pods by executing ```kubectl get pods``` and wait until you have all your ClearML pods "Running".  
Example output:  
```
[rjuarez@controlplane ~]$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS        AGE
clearml-apiserver-845544cd9f-sxr8n               1/1     Running   1 (7m13s ago)   21h
clearml-apiserver-asyncdelete-67b6b746bb-gbbhx   1/1     Running   1 (7m13s ago)   21h
clearml-elastic-master-0                         1/1     Running   1 (7m13s ago)   102m
clearml-fileserver-85cd59dbdb-68brt              1/1     Running   1 (7m13s ago)   21h
clearml-mongodb-0                                1/1     Running   5 (7m13s ago)   21h
clearml-mongodb-1                                1/1     Running   5 (7m13s ago)   21h
clearml-redis-master-0                           1/1     Running   4 (7m13s ago)   21h
clearml-redis-replicas-0                         1/1     Running   8 (5m40s ago)   21h
clearml-redis-replicas-1                         1/1     Running   8 (5m36s ago)   21h
clearml-webserver-cf8c779bd-f9js4                1/1     Running   1 (7m13s ago)   21h
```  

Finally, use the URL set previously in `values-production.yaml` (e.g.: http://app.clearml.172-16-10-11.nip.io/) in your web browser to check if your ClearML app is successfully running.


## 3. ClearML Agent installation

In order to be able to schedule and run jobs, you must install the ClearML Agent.  

### 3.1. Create API Credentials
First you need to create your API credentials. Open the ClearML web interface and log in. Click on your user icon in the top right corner, navigate to **Settings** > **Workspace**, and click on **"Create new credentials"**. Copy the Access Key and Secret Key provided. These credentials will allow the agent to authenticate with the ClearML server.  

### 3.2. Configure your Agent deployment
Create a `values-agent.yaml` file and configure it using the credentials you got in the previous step:  
```yaml
clearml:
  agentk8sglueKey: "YOUR_ACCESS_KEY"
  agentk8sglueSecret: "YOUR_SECRET_KEY"

agentk8sglue:
  apiServerUrlReference: "YOUR_APISERVER_URL"
  fileServerUrlReference: "YOUR_FILESERVER_URL"
  webServerUrlReference: "YOUR_WEBSERVER_URL"
  queue: default
```
For the URL references, it is highly recommended to use the internal Kubernetes service names and ports instead of the public ingress domains. This avoids connection issues. For example:  
```yaml
agentk8sglue:
  apiServerUrlReference: "http://clearml-apiserver:8008"
  fileServerUrlReference: "http://clearml-fileserver:8081"
  webServerUrlReference: "http://clearml-webserver:8080"
  queue: default
```
*NOTE: you may edit the template provided by ClearML (`clearml-helm-charts/charts/clearml-agent/values.yaml`) instead of creating a new YAML file.*  

### 3.3. Install the Agent
Install the agent using Helm:  
```bash
helm install clearml-agent clearml/clearml-agent -f values-agent.yaml
```   

### 3.4. Verifying the installation
Finally, check if the Agent is already running:  
```bash
kubectl get pods
```  
If your ClearML-Agent pod is in "Running" state, you should now be able to schedule and execute your jobs.  

#### Troubleshooting: Agent Pod in Error State
If your `clearml-agent` pod starts running but suddenly changes its state to "Error", you may have found a bug.  
You can fix it by replacing the `agentk8sglue` block in your `values-agent.yaml` with this configuration, which disables the auto-update behavior:  
```yaml
agentk8sglue:
  apiServerUrlReference: "http://clearml-apiserver:8008"
  fileServerUrlReference: "http://clearml-fileserver:8081"
  webServerUrlReference: "http://clearml-webserver:8080"
  queue: default
  extraEnvs:
    - name: CLEARML_AGENT_NO_UPDATE
      value: "true"
```  
Update the installation:  
```bash
helm uninstall clearml-agent
helm install clearml-agent clearml/clearml-agent -f values-agent.yaml
```  

Finally, check `kubectl get pods` again to verify if the Agent is running properly.  
