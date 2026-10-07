# kubernetes-RBAC

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

2) Now create service and clusterRoleBinding

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

3) Get the Access Token Retrieve the token for the admin-user:

    kubectl -n kubernetes-dashboard create token admin-user

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


