# edge_anomaly_detection_demo

## Prepare your own instance of the gitops repository

Fork [hhttps://github.com/nexus-Six/edge_anomaly_detection_demo.git](hhttps://github.com/nexus-Six/edge_anomaly_detection_demo.git) into your own Git repo (you will need to make changes in this repo).

Clone the forked repo to your local home directory 
```bash
cd ~

git clone <your_git_organisation>/edge_anomaly_detection_demo.git

```

Replace the git repo paths in
- config/instances/manuela-quickstart/manuela-quickstart-line-dashboard-application.yaml
-  config/instances/manuela-quickstart/manuela-quickstart-messaging-application.yaml
-  config/instances/manuela-quickstart/manuela-quickstart-machine-sensor-application.yaml

Replace the OCP paths in
- config/instances/manuela-quickstart/line-dashboard/line-dashboard-configmap-config.json
- config/instances/manuela-quickstart/line-dashboard/line-dashboard-route.yaml
- config/instances/manuela-quickstart/machine-sensor/machine-sensor-1-configmap.properties
- config/instances/manuela-quickstart/machine-sensor/machine-sensor-2-configmap.properties 
- config/instances/manuela-quickstart/messaging/route.yaml 


Push the changes

```bash
git add .

git commit -m "adapt repo URLs"

git push
```

## Deploy the OpenShift GitOps (ArgoCD) Operator

Installing the OpenShift GitOps Operator is a manual step in this quickstart guide.

Follow the OpenShift Documentation. I.e. [Installing OpenShift GitOps Operator in web console](https://docs.openshift.com/container-platform/4.10/cicd/gitops/installing-openshift-gitops.html#installing-gitops-operator-in-web-console_installing-openshift-gitops) 

Please select:
- Latest update channel
- Installation mode: All namespaces
- Installed Namespaces: openshift-operators
- Update approval: Automatic 


Wait until the OpenShift GitOps operator is installed. This might take a short while. 

After the Red Hat OpenShift GitOps Operator is installed, it automatically sets up a ready-to-use Argo CD instance that is available in the openshift-gitops namespace, and an Argo CD icon is displayed in the console toolbar. 


After the installation is complete, ensure that all the pods in the openshift-gitops namespace are running:
```
oc get pods -n openshift-gitops

```
Example output:
```
NAME                                                      	READY   STATUS	RESTARTS   AGE
cluster-b5798d6f9-zr576                                   	1/1 	Running   0      	65m
kam-69866d7c48-8nsjv                                      	1/1 	Running   0      	65m
openshift-gitops-application-controller-0                 	1/1 	Running   0      	53m
openshift-gitops-applicationset-controller-6447b8dfdd-5ckgh 1/1 	Running   0      	65m
openshift-gitops-redis-74bd8d7d96-49bjf                   	1/1 	Running   0      	65m
openshift-gitops-repo-server-c999f75d5-l4rsg              	1/1 	Running   0      	65m
openshift-gitops-server-5785f7668b-wj57t                  	1/1 	Running   0      	53m
```



Note, in this quickstart deployment, we will use the cluster Argo CD instance and won't deploy any Manuela specific Argo CD instance.


Check that you can login into the Argo CD instance by using the Argo CD admin account:
- Follow the instructions in the documentation: [Logging in to the Argo CD instance by using the Argo CD admin account](https://docs.openshift.com/container-platform/4.10/cicd/gitops/installing-openshift-gitops.html#logging-in-to-the-argo-cd-instance-by-using-the-argo-cd-admin-account_installing-openshift-gitops)
- Or, pull the `admin` password with 
  
```
oc get secret openshift-gitops-cluster -o 'go-template={{index .data "admin.password"}}' -n openshift-gitops |  base64 -d
```

The admin credentials are needed because the manual sync runs into problems, when you login into ArgoCD with an OpenShift user.  


## Deploy the Applications via GitOps

We will deploy the three part of the core Manuele Applicatoion step by step:

1. Messaging component inclusive the AMQ Broker Operator and instance
2. Line Dashboard
3. Senor Simulators 

### Install the messaging component inclusive the AMQ Broker Operator and instance

Deploy the Messaging component:

```
$ oc apply -f config/instances/manuela-quickstart/manuela-quickstart-messaging-application.yaml
```
Example output:
```
application.argoproj.io/manuela-quickstart-messaging created
```

Check the status:
```
$ oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-messaging        Synced        Progressing

```

Use the Argo CD UI to follow the installation.

You might run into issues with the AMQ operator installation when this guide a a bit outdated. Check with the OpenShift Administrator Console, if the update chancel, that is defined in `config/templates/manuela/messaging/amq-operator-subscription.yaml` is still available for the AMQ operator.




### Install the line dashboard component 

Deploy the line dashboard component

```
$ oc apply -f config/instances/manuela-quickstart/manuela-quickstart-line-dashboard-application.yaml
```
Example output:
``` 
application.argoproj.io/manuela-quickstart-line-dashboard created

```

Check the status:
```
$ oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-line-dashboard   Synced        Progressing
manuela-quickstart-messaging        Synced        Healthy
```

Wait until the application is `Healthy`.


Open the Manuela line dashboard web application and view the sensor data by selecting the "Realtime Data" menu item. Retrieve the UI URL via:
```bash
echo http://$(oc -n manuela-quickstart-line-dashboard get route line-dashboard -o jsonpath='{.spec.host}')/sensors
```

Note, no sensor data is displayed , because the senor simulators are not deployed yet.

### Install the Senor Simulators components 

Deploy the line machine sensor application:

```
$ oc apply -f config/instances/manuela-quickstart/manuela-quickstart-machine-sensor-application.yaml
```
Example output:
``` 
application.argoproj.io/manuela-quickstart-machine-sensor created
```

Wait until the application is `Healthy`.

```
$ oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-line-dashboard   Synced        Healthy
manuela-quickstart-machine-sensor   Synced        Healthy
manuela-quickstart-messaging        Synced        Healthy
```

### Deploy the Anomaly Detection ML Service

- Install the OpenShift DataScience Operator from the Operatorhub with default settings
- After installation open the OpenShift Datascience Dasboard by clicking on the link provided at the top right menu under the square icon
- Log in
- In the menu on the left  