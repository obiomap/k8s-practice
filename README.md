# k8s-practice

## k8s Cluster Installation
##### Prerequisites
1. You need a Google Cloud Platform account with billing enabled. Visit the Google Developers Console for more details.
2. Install gcloud as necessary. gcloud can be installed as a part of the Google Cloud SDK.
3. Enable the Compute Engine Instance Group Manager API in the Google Cloud developers console.
4. Make sure that gcloud is set to use the Google Cloud Platform project you want. You can check the current project using gcloud config list project and change it via gcloud config set project <project-id>.
5. Make sure you have credentials for GCloud by running gcloud auth login.
6. (Optional) In order to make API calls against GCE, you must also run gcloud auth application-default login.
7. Make sure you can start up a GCE VM from the command line. 

##### Starting a cluster

You can install a client and start a cluster with either one of these commands (we list both in case only one is installed on your machine):
```
	curl -sS https://get.k8s.io | bash
	or
	wget -q -O - https://get.k8s.io | bash
```	
run the <kubernetes>/cluster/kube-up.sh script to start the cluster:
	```
	cd kubernetes
	cluster/kube-up.sh
	```
Verify cluster using kubectl command
	```kubectl cluster-info```
	
## Deploy Guest Application developemt namespace
Create development namespace
```kubectl create namespace deployment```
##### Install Redis master
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml -n development
kubectl get pods -n development
kubectl logs -f POD-NAME -n development
```
#### install redis master service
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml -n development
kube get svc -n development
```
##### install redis salve
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml -n development
kubectl get pods -n development
```
##### install redis slave service
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml -n development
```
##### install guestbook
```
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml -n development
kubectl get pods -l app=guestbook -l tier=frontend -n development
```
##### install guest book service
```
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml -n development
```
##### make frontend-service to loadbalance instead of Node-port, edit file and change type: LoadBalancer
```
kubectl edit svc frontend -n development
kubectl get services -n development
```
##### Copy the external IP address, and load the page in your browser to view your guestbook.

 scaling up frontend
 ```
 kubectl scale deployment frontend --replicas=5
 ```

## Install and configure helm
Install Helm CLI tool using followig command:
```
	cd ~/environment
	curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
	chmod +x get_helm.sh
	./get_helm.sh
```
Configure Helm access with RBAC

	Helm relies on a service called tiller that requires special permission on the kubernetes cluster, so we need to build a Service Account for tiller to use. We’ll then apply this to the cluster.
	
	To create a new service account manifest:
```
cat <<EoF > helm-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF
```
	
```kubectl apply -f helm-rbac.yaml```

Verify helm version
```helm version```

## Install Jenkins on Centos 7
	```
		# install JAVA
		sudo yum install java-1.8.0-openjdk-devel
		#Configure YUM Repo
		curl --silent --location http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo | sudo tee /etc/yum.repos.d/jenkins.repo
		sudo rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
		# install jenkins
		sudo yum install jenkins
		# configure jenkins service
		sudo systemctl start jenkins
		sudo systemctl status jenkins
		sudo systemctl enable jenkins
		# Configure Firewall rules
		sudo firewall-cmd --permanent --zone=public --add-port=8080/tcpsudo 
		firewall-cmd --reload
	```
## Application CI/CD using Helm
Please refer follwoing jenkins file
https://github.com/anandnevase/bday-app/blob/master/jenkinsfile-k8s

Detail explanation of application and its CI/CD process mention in following link:
https://github.com/anandnevase/bday-app

## EFK configuration

##### Deploy Elasticsearch

```
kubectl create ns logging
```
Helm installation of Elasticsearch. We will disable persistence for simplicity. Warning this will consume a lot of memory in your cluster.

```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm install --name elasticsearch incubator/elasticsearch \
    --set master.persistence.enabled=false \
    --set data.persistence.enabled=false \
    --set image.tag=6.4.2 \
    --namespace logging
```

##### Deploy Kibana
If you used Elasticsearch deployment 
```
helm install --name kibana stable/kibana \
    --set env.ELASTICSEARCH_URL=http://elasticsearch-client:9200 \
    --set image.tag=6.4.2 \
    --namespace logging
```

##### Deploy Fluent Bit

** Create the RBAC resources for Fluent Bit **
```
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml

kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml

kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml
```

** Create the Fluent Bit Config Map **
```
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-configmap.yaml
```

** Deploy the Fluent Bit DaemonSet **
```
wget https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml
# modify as recommended, then:
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml
```

** Check if everything is running **
```
kubectl get pods -n logging
NAME                             READY     STATUS    RESTARTS   AGE
elasticsearch-78987949dc-7wj8m   1/1       Running   0          1d
fluent-bit-2dv5n                 1/1       Running   7          1d
kibana-6f75b4fdcf-9qbp7     
```

** Populate logs **
Deploy an example Nginx container and port-forward the traffic to your localhost.
```
kubectl run nginx --image=nginx -n logging

kubectl port-forward nginx-8586cf59-kpbf6 8081:80 &
```
Curl it a few times, and press Ctrl+C when done.
```
while true; do curl localhost:8081; sleep 2; done
```

** Viewing Logs in Kibana **
Access Kibana quickly through port-forwarding
```
kubectl port-forward kibana-6f75b4fdcf-9qbp7 5601
```
	
## Blue Green deployment

create Blue-Green namespace
```
kubectl create ns blue-green
```
Deploy Blue version of application
```
DEPLOYMENT=blue IMAGE="hanzel/nginx-html" IMAGE_TAG="1" APP="nginx" envsubst < blue-green-deployment/deployment.yaml | kubectl apply -n blue-green -f -
DEPLOYMENT=blue APP="nginx" envsubst < blue-green-deployment/service.yaml | kubectl apply -n blue-green -f -
```

 Deploy Green Version of application
```
DEPLOYMENT=green IMAGE="hanzel/nginx-html" IMAGE_TAG="2" APP="nginx" envsubst < blue-green-deployment/deployment.yaml | kubectl apply -n blue-green -f -
DEPLOYMENT=green APP="nginx" envsubst < blue-green-deployment/service.yaml | kubectl apply -n blue-green -f -
```
Switching between Blue and Green Deployments:
Change the “selector -> color” from “blue” to “green”. Save the file.
```
kubectl edit -n blue-green service nginx-blue
```

## Canary deployment

 create Blue-Green namespace
```
kubectl create ns canary
```
Deploy prod version application
```
ENV=prod IMAGE="hanzel/nginx-html" IMAGE_TAG="1" APP="nginx" envsubst < canary-deployment/deployment.yaml | kubectl apply -n canary -f -
```

deploy canary version
```
ENV=canary IMAGE="hanzel/nginx-html" IMAGE_TAG="2" APP="nginx" envsubst < canary-deployment/deployment.yaml | kubectl apply -n canary -f -
```

## Monitoring
Create Monitoring namespace
```kubectl create ns monitoring```

##### First we need to update our local helm chart repo.
```$ helm repo update```

##### Install Prometheus into the monitoring namespace
```
$ helm install stable/prometheus \
    --namespace monitoring \
    --name prometheus
```
This will deploy Prometheus into your cluster in the monitoring namespace and mark the release with the name prometheus.
Prometheus is now scraping the cluster together with the node-exporter and collecting metrics from the nodes.
We can confirm by checking that the pods are running:

```kubectl get pods -n monitoring```

##### Install Grafana
```
kubectl apply -f monitoring/grafana/config.yml

helm install stable/grafana \
    -f monitoring/grafana/values.yml \
    --namespace monitoring \ 
    --name grafana
```	
GET Grafana PASSWORD:
```
kubectl get secret \
    --namespace monitoring grafana \
    -o jsonpath="{.data.admin-password}" \
    | base64 --decode ; echo
```


Port Forward the Grafana dashboard to see whats happening:
```
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
$ kubectl --namespace monitoring port-forward $POD_NAME 3000
```
1. Login to Grafana using admin credentials
2. Add a dashboard
3. In the left hand menu, choose Dashboards > Manage > + Import
4. In the Grafana.com dashboard input, add the dashboard ID we want to use: 1860 and click Load
5. On the next screen select a name for your dashboard and select Prometheus as the datasource for it and click Import.
6. You’ve got metrics !