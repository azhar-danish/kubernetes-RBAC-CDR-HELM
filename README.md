# kubernetes-RBAC
----------------------------------Role Based Access Control ---------------------------------
1) create a role

role.yml

---------------

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: 
  namespace: apache
  name: apache-manager
rules:
- apiGroups: ["","apps", "extensions","batch"]
  resources: ["pods","deployments", "replicasets", "jobs","services"]
  verbs: ["get", "watch", "list", "create", "update", "patch", "delete","apply"]

k apply -f role.yml
---------------

2) create as service Account

service-account.yml

--------------------
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache  


k apply -f servic-account.yml
--------------------

3) 

azhardanish@Mac kubernetes-rbac % kubectl auth whoami

ATTRIBUTE                                           VALUE
Username                                            kubernetes-admin
Groups                                              [kubeadm:cluster-admins system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=2957db86b8738808c013eb33fb1c29041b559e3906877cefc9d143c37691f644]

4)  kubectl auth can-i create pod
yes

5) k auth can-i get pods -n apache
yes

6) azhardanish@Mac kubernetes-rbac % k auth can-i get pods -n apache --as=apache-user
no

7) Create Role-binding

    role-binding.yml
    --------------------------------
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: apache-manager-binding
    namespace: apache
    subjects:
    - kind: User
        name: apache-user
        apiGroup: rbac.authorization.k8s.io
    roleRef:
        kind: Role
        name: apache-manager
        apiGroup: rbac.authorization.k8s.io

    k apply -f role-binding.yml
    --------------------------------

8) k auth can-i get pods -n apache --as=apache-user
yes

---------------------Cluster level Role Binding--------------------------


1) kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

it will create role and different resource

2) Now create serviceAccount and clusterRoleBinding

admin-user-dashboard.yml
------------------------------
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard 

---
#cluster role binding for the admin-user service account
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: cluster-admin

k apply -f admin-user-dashboard.yml

------------------------------
k get serviceAccount -n kubernetes-dashboard

NAME                   AGE
admin-user             16h
default                16h
kubernetes-dashboard   16h



k get clusterRoleBinding -n kubernetes-dashboard | grep "admin-user"
admin-user           ClusterRole/cluster-admin                                                          16h


azhardanish@Mac dashboard % 



3) Get the Access Token Retrieve the token for the admin-user:

    kubectl create token admin-user -n kubernetes-dashboard

    Copy the token for use in the Dashboard login.

Access the Dashboard Start the Dashboard using kubectl proxy:

kubectl proxy
Open the Dashboard in your browser:

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

for server : kubectl proxy --port=8001 --address=0.0.0.0 --accept-hosts='.*'


-------------Custom Resource Definition---------------------------------

1) Create a CRD yml

devops-crd.yml
---------------------
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: devops.azhargroup.com # must be in the form of <plural>.<group>
spec:
  group: azhargroup.com 
  names:
    plural: devops  
    singular: devop
    kind: DevOps
    shortNames:
      - dev
      - dops
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties: 
            spec:
              type: object
              properties:
                name:
                  type: string
                  description: "Name of the DevOps batch"
                duration:
                  type: string
                  description: "Duration of the DevOps batch"
                mode:
                  type: string
                  description: "Mode of the DevOps batch"
                platform:
                  type: string
                  description: "Platform of the DevOps batch" 


k apply -f devops-crd.yml
--------------------
2) 
azhardanish@Mac CDR % k get crd

NAME                                                  CREATED AT
devops.azhargroup.com                                 2026-10-07T17:43:19Z
verticalpodautoscalercheckpoints.autoscaling.k8s.io   2026-10-07T10:49:34Z
verticalpodautoscalers.autoscaling.k8s.io             2026-10-07T10:49:34Z

3) Create new custom-resource

custom-resource.yml
------------------------

apiVersion: azhargroup.com/v1
kind: DevOps
metadata:
  name: my-devops-batch
spec:
  name: My DevOps Batch
  duration: 30 days, Mon-Fri, 9am-12pm
  mode: live
  platform: Kubernetes 

  k apply -f custom-resource.yml
------------------------

azhardanish@Mac CDR % kubectl get DevOps
NAME              AGE
my-devops-batch   3m10s
azhardanish@Mac CDR % 


-------------------------HELM ----------------------------------

1) Installing helm

> mkdir helm
> cd helm
> curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
> chmod 700 get_helm.sh
> ./get_helm.sh

> helm version
version.BuildInfo{Version:"v4.3.0", GitCommit:"bec5b06ed841fe5269972d864d5177944fd5970f", GitTreeState:"clean", GoVersion:"go1.27.1", KubeClientVersion:"v1.37"}

creating helm chart

syntax: helm create <project-name>

> helm create apache-helm
Creating apache-helm

> helm > ls -lh
drwxr-xr-x  7 azhardanish  staff   224B  8 Oct 00:51 apache-helm
-rwx------  1 azhardanish  staff    12K  8 Oct 00:45 get_helm.sh

> helm > cd apache-helm 

> apache-helm > ls -lh

-rw-r--r--   1 azhardanish  staff   1.1K  8 Oct 00:51 Chart.yaml
drwxr-xr-x   2 azhardanish  staff    64B  8 Oct 00:51 charts
drwxr-xr-x  11 azhardanish  staff   352B  8 Oct 00:51 templates
-rw-r--r--   1 azhardanish  staff   5.2K  8 Oct 00:51 values.yaml


tree
.
├── Chart.yaml
├── charts
├── templates
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── httproute.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml


open  values.yaml using 

vi values.yaml 

save it using esc + !wq

update the variable like 
--image: httpd
port: 80
targetPort : 80
replicas: 2

> apache-helm > cd ..
> helm > 

package the helm with edited values

syntax : helm package <project-name>
eg:-
> helm > helm package apache-helm/


Successfully packaged chart and saved it to: /Users/azhardanish/Downloads/kubernetes/kubernetes-rbac/helm/apache-helm-0.1.0.tgz


> helm > ls -lh
total 40
drwxr-xr-x  7 azhardanish  staff   224B  8 Oct 01:10 apache-helm
-rw-r--r--  1 azhardanish  staff   4.9K  8 Oct 01:11 apache-helm-0.1.0.tgz (newly created)
-rwx------  1 azhardanish  staff    12K  8 Oct 00:45 get_helm.sh

installing helm

syntax : helm install <resource-name> <project-name>

eg:

> helm > helm install dev-apache apache-helm

NAME: dev-apache
LAST DEPLOYED: Thu Oct  8 01:15:15 2026
NAMESPACE: apache
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace apache -l "app.kubernetes.io/name=apache-helm,app.kubernetes.io/instance=dev-apache" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace apache $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace apache port-forward $POD_NAME 8080:$CONTAINER_PORT
azhardanish@Mac helm % 


Creating helm-chart with a new namespace

syntax: helm install <release-name> <project-name> -n <new-namespace-name> --create-namespace

eg:

helm install dev-apache apache-helm -n dev-apache --create-namespace

NAME: dev-apache
LAST DEPLOYED: Thu Oct  8 01:21:38 2026
NAMESPACE: dev-apache
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace dev-apache -l "app.kubernetes.io/name=apache-helm,app.kubernetes.io/instance=dev-apache" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace dev-apache $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace dev-apache port-forward $POD_NAME 8080:$CONTAINER_PORT
azhardanish@Mac helm % 




For removing dev-apache

helm list -A
NAME    	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART                 	APP VERSION
dev-apache	dev-apache 	1       	2026-10-08 11:28:31.673093 +0530 IST	deployed	apache-helm-chart-0.1.0	1.16.0  

helm uninstall dev-apache -n dev-apache

helm unistall dev-apache


helm create
helm package
helm install
helm uninstall


Watn to update the the release

1) cd helm
   helm > cd node-app
   helm > node-app > vi values.yml
   like
   replicas: 2 to replicas: 3
   save => esc + :wq

   helm > node-app > vi Chart.yaml
   change appVersion: 16.1.1
   save => esc + :wq

2) helm > helm package node-app

3) helm list -A 
NAME         	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART         	APP VERSION
node-app-prod	development	1       	2026-10-08 13:39:33.401349 +0530 IST	deployed	node-app-0.1.0	1.16.0     



4) syntax: 
        helm upgrade <release-name> <project-name> -n <new-namespace-name>

    eg:
        helm upgrade node-app-prod node-app -n development

5) helm Rollback

helm list -A
NAME         	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART         	APP VERSION
node-app-prod	development	2       	2026-10-08 13:46:02.578646 +0530 IST	deployed	node-app-0.1.0	1.16.1 

helm rollback node-app-prod 1 -n development
Rollback was a success! Happy Helming!


Installing a Repo from Artifact hub (https://artifacthub.io/)

search  a repo like : Nginx

select any repo like cloudpirates


> helm install nginx oci://registry-1.docker.io/cloudpirates/nginx 
> k get po
NAME                             READY   STATUS    RESTARTS   AGE
my-nginx-5fdb556678-6z8mz        1/1     Running   0          50s (running)
node-app-prod-5cd486675d-6t6tl   1/1     Running   0          35m

> helm list -A
NAME         	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART         	APP VERSION
my-nginx     	development	1       	2026-10-08 14:31:57.329461 +0530 IST	deployed	nginx-0.16.12 	1.31.6     
node-app-prod	development	3       	2026-10-08 13:57:03.759773 +0530 IST	deployed	node-app-0.1.0	1.16.0     

helm uninstall my-nginx

-------------------------------Installing mongodb using helm repo----------------------------------

> helm install my-mongodb oci://registry-1.docker.io/cloudpirates/mongodb

Pulled: registry-1.docker.io/cloudpirates/mongodb:0.20.0
Digest: sha256:38a6c56132badd2897bdc2999a5e76111f35af5315b2c6cd91279bbaf9e1f8e9
NAME: my-mongodb
LAST DEPLOYED: Thu Oct  8 14:40:37 2026
NAMESPACE: development
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

> helm list -A                                                           
NAME      	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART         	APP VERSION
my-mongodb	development	1       	2026-10-08 14:40:37.151396 +0530 IST	deployed	mongodb-0.20.0	9.0.2      


k get secret my-mongodb -n development -o jsonpath="{.data.mongodb-root-password}" | base64 --decode

k exec -it my-mongodb-0 -n development -- mongosh -u admin -p ZbZpLkt8cVWqbKeD

enter in the shell









