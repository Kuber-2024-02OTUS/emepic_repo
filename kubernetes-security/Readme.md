# Создаем новую ветку kubernetes-security и переходим в нее
git checkout -b kubernetes-security

# Создаем новую папку kubernetes-security
mkdir kubernetes-security && cd kubernetes-security

# Создаем манифест namespace.yaml и копируем в него данные из предыдущего ДЗ
touch namespace.yaml

# Применяем манифест namespace.yaml
kubectl apply -f namespace.yaml
namespace/homework created

# Проверяем что namespace создался
nickolayfilippov@MacBook kubernetes-security % kubectl get ns
NAME              STATUS   AGE
default           Active   101d
homework          Active   13s
ingress-nginx     Active   33d
kube-node-lease   Active   101d
kube-public       Active   101d
kube-system       Active   101d

# Создаем манифест service-account.yaml для создания service account monitoring
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring
  namespace: homework

# Применяем манифест service-account.yaml
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f service-account.yaml
serviceaccount/monitoring created

# Проверяем что service account monitoring создался
nickolayfilippov@MacBook kubernetes-security % kubectl get sa -n homework
NAME         SECRETS   AGE
default      0         3m52s
monitoring   0         52s

# Cоздаем манифест role.yaml для назначения доступа к эндпоинту /metrics кластера
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

# Применяем манифест role.yaml
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f role.yaml
clusterrole.rbac.authorization.k8s.io/metrics-reader created

# Проверяем что роль создалась
nickolayfilippov@MacBook kubernetes-security % kubectl get clusterroles

NAME                                                                   CREATED AT
admin                                                                  2024-03-08T03:34:12Z
cluster-admin                                                          2024-03-08T03:34:12Z
edit                                                                   2024-03-08T03:34:12Z
ingress-nginx                                                          2024-05-14T13:55:54Z
ingress-nginx-admission                                                2024-05-14T13:55:54Z
kubeadm:get-nodes                                                      2024-03-08T03:34:13Z
metrics-reader                                                         2024-06-17T06:58:40Z
system:aggregate-to-admin                                              2024-03-08T03:34:12Z
system:aggregate-to-edit                                               2024-03-08T03:34:12

nickolayfilippov@MacBook kubernetes-security % kubectl describe clusterrole metrics-reader

Name:         metrics-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  endpoints  []                 []              [get list watch]
  nodes      []                 []              [get list watch]
  pods       []                 []              [get list watch]
  services   []                 []              [get list watch]
             [/metrics]         []              [get]


# Создаем манифест cluster-role-binding.yaml, для связи роли с service account monitoring
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-metrics-binding
subjects:
- kind: ServiceAccount
  name: monitoring
  namespace: homework
roleRef:
  kind: ClusterRole
  name: metrics-reader
  apiGroup: rbac.authorization.k8s.io

# Применяем манифест cluster-role-binding.yaml
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f cluster-role-binding.yaml
clusterrolebinding.rbac.authorization.k8s.io/monitoring-metrics-binding created

# Проверяем clusterrolebindings
nickolayfilippov@MacBook kubernetes-security % kubectl get clusterrolebindings
NAME                                                   ROLE                                                                               AGE
cluster-admin                                          ClusterRole/cluster-admin                                                          101d
ingress-nginx                                          ClusterRole/ingress-nginx                                                          33d
ingress-nginx-admission                                ClusterRole/ingress-nginx-admission                                                33d
kubeadm:get-nodes                                      ClusterRole/kubeadm:get-nodes                                                      101d
kubeadm:kubelet-bootstrap                              ClusterRole/system:node-bootstrapper                                               101d
kubeadm:node-autoapprove-bootstrap                     ClusterRole/system:certificates.k8s.io:certificatesigningrequests:nodeclient       101d
kubeadm:node-autoapprove-certificate-rotation          ClusterRole/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   101d
kubeadm:node-proxier                                   ClusterRole/system:node-proxier                                                    101d
minikube-rbac                                          ClusterRole/cluster-admin                                                          101d
monitoring-metrics-binding                             ClusterRole/metrics-reader                                                         82s
storage-provisioner                                    ClusterRole/system:persistent-volume-provisioner                                   101d

nickolayfilippov@MacBook kubernetes-security % kubectl describe clusterrolebinding monitoring-metrics-binding

Name:         monitoring-metrics-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  metrics-reader
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  monitoring  homework

# Копируем манифесты из предыдущего ДЗ и изменяем deployment.yaml, чтобы поды запускались под service account monitoring
spec:
      serviceAccountName: monitoring

# Применяем манифесты и проверяем что поды запустились и запущены под sa monitoring
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f pvc.yaml
persistentvolumeclaim/pvchw4 created
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f cm.yaml 
configmap/cmhw4 created
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f deployment.yaml 
deployment.apps/controllers-pod created
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f service.yaml 
service/webapp-service created
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/webapp-ingress created
nickolayfilippov@MacBook kubernetes-security % kubectl get po -n homework
NAME                               READY   STATUS    RESTARTS   AGE
controllers-pod-5fcbb7dbfb-7psck   1/1     Running   0          21s
controllers-pod-5fcbb7dbfb-p7z8q   1/1     Running   0          21s
controllers-pod-5fcbb7dbfb-swmsb   1/1     Running   0          21s

nickolayfilippov@MacBook kubernetes-security % kubectl describe pod controllers-pod-5fcbb7dbfb-7psck -n homework
Name:             controllers-pod-5fcbb7dbfb-7psck
Namespace:        homework
Priority:         0
Service Account:  monitoring
Node:             minikube/192.168.49.2
Start Time:       Mon, 17 Jun 2024 10:17:40 +0300
Labels:           app=webapp
                  pod-template-hash=5fcbb7dbfb

# Создаем манифест service-account-cd.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cd
  namespace: homework

# Применяем манифест service-account-cd.yaml и проверяем что sa создался
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f service-account-cd.yaml
serviceaccount/cd created
nickolayfilippov@MacBook kubernetes-security % kubectl get serviceaccount cd -n homework
NAME   SECRETS   AGE
cd     0         36s

# Создаем манифест rolebinding-cd-admin.yaml для связи sa c ролью admin
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cd-admin-binding
  namespace: homework
subjects:
- kind: ServiceAccount
  name: cd
  namespace: homework
roleRef:
  kind: Role
  name: admin
  apiGroup: rbac.authorization.k8s.io

# Создаем и применяем манифест admin-role.yaml для создания роли admin с полными правами на ns homework
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: homework
  name: admin
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]

nickolayfilippov@MacBook kubernetes-security % kubectl apply -f admin-role.yaml
role.rbac.authorization.k8s.io/admin created

# Применяем манифест rolebinding-cd-admin.yaml и проверяем
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f rolebinding-cd-admin.yaml
rolebinding.rbac.authorization.k8s.io/cd-admin-binding created

nickolayfilippov@MacBook kubernetes-security % kubectl get rolebinding cd-admin-binding -n homework
NAME               ROLE         AGE
cd-admin-binding   Role/admin   20s

nickolayfilippov@MacBook kubernetes-security % kubectl describe rolebinding cd-admin-binding -n homework
Name:         cd-admin-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  admin
Subjects:
  Kind            Name  Namespace
  ----            ----  ---------
  ServiceAccount  cd    homework

# Создаем временный токен для cd и направляем его в файл token
kubectl create token cd -n homework --duration=24h > token

# Получаем токен, сертификат и API сервера
nickolayfilippov@MacBook kubernetes-security % kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
https://127.0.0.1:64537

# Создаем kubeconfig-cd.yaml на основе полученных данных
apiVersion: v1
clusters:
- cluster:
    server: https://127.0.0.1:64537
  name: kubernetes
- cluster:
    certificate-authority: /Users/nickolayfilippov/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Mon, 17 Jun 2024 11:09:08 MSK
        provider: minikube.sigs.k8s.io
        version: v1.32.0
      name: cluster_info
    server: https://127.0.0.1:64537
  name: minikube
contexts:
- context:
    cluster: kubernetes
    namespace: homework
    user: cd
  name: cd@kubernetes
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Mon, 17 Jun 2024 11:09:08 MSK
        provider: minikube.sigs.k8s.io
        version: v1.32.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: cd
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IjRreElEdWo4SmROVVpkaXJzVzRZZFlrNnQ2eV92YjlEV0ljTkwtcjF6WUEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzE4Njk3OTUzLCJpYXQiOjE3MTg2MTE1NTMsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJob21ld29yayIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJjZCIsInVpZCI6IjdmN2ZiN2UyLTA0NTAtNDlkMS05ZTFkLTE5ZjU0ZDMxNGU1ZCJ9fSwibmJmIjoxNzE4NjExNTUzLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6aG9tZXdvcms6Y2QifQ.X2icdrOu2aHauBtDszPy2vbukAFUgNPaDCOI7VeXtDcqxNXt5a-aTVE6ROY661wx8FKScF3lKtxwX4v10lqrJo_6_UBA2TKnPl9JpETDraBIF3YsIZClXPnEXv_NCzFFJcDY6stDCnhZ8mF21sKJOFWexIslqxCEX3HfUI329k9yc1w582S3hN147ArODGWXjNc0mxFO-i2gZqX04on5eU6ClHo7m2TcH4fIpkkVoqcmChhYT7dh42q1jzIzgv_XesINgt3wnoTnhGA1yrRBHa15xemv_W9UL4BA0Fd2j_tdvY8RdN6h4m3eGtxAXmbAorxaHjSMXhLClWAuiZGLuA
- name: minikube
  user:
    client-certificate: /Users/nickolayfilippov/.minikube/profiles/minikube/client.crt
    client-key: /Users/nickolayfilippov/.minikube/profiles/minikube/client.key

# Подключаемся к minikube при помощи kubeconfig-cd.yaml и проверяем права
nickolayfilippov@MacBook kubernetes-security % export KUBECONFIG=kubeconfig-cd.yaml
ickolayfilippov@MacBook kubernetes-security % kubectl get pods -n homework      
NAME                               READY   STATUS    RESTARTS        AGE
controllers-pod-5fcbb7dbfb-7psck   1/1     Running   2 (6m15s ago)   57m
controllers-pod-5fcbb7dbfb-p7z8q   1/1     Running   2 (6m15s ago)   57m
controllers-pod-5fcbb7dbfb-swmsb   1/1     Running   2 (6m15s ago)   57m

# Задание со *

# Генерируем токен для monitoring
kubectl create token monitoring -n homework
eyJhbGciOiJSUzI1NiIsImtpZCI6IjRreElEdWo4SmROVVpkaXJzVzRZZFlrNnQ2eV92YjlEV0ljTkwtcjF6WUEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzE4NjE4MzQ1LCJpYXQiOjE3MTg2MTQ3NDUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJob21ld29yayIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJtb25pdG9yaW5nIiwidWlkIjoiYTVjYTQ2ZGYtOTRhZC00YThmLWI4MmItZDhmZDVhOWU4NGQ1In19LCJuYmYiOjE3MTg2MTQ3NDUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpob21ld29yazptb25pdG9yaW5nIn0.XOA1V926NzOKCHsCVGo6t340LaM29PviQ3P4Dp8y2FXy_wA28yiomFwttJcwc1ujgCLerrQksLf0qlJaWUDgpGn4-BUcFMgEFDXUf7ubHXhbFZvFn4JB0u-7rTup7I9B_Tn2YRvB0BLza-SJ-yihGY2Syy2_eY7GpsA8mf3RsFfzgIX_L07xVDU6jppE_ME3iXQtOM_9dQe3Z3Y-PzLAdCndNUnKzXrzSOd2KwMLHAApJpayN__MUpMrGqqSsh-DDqDgnuvjnPrJHj-J5GtpGqkAB5kfq6Pb5qNIX8kpUKyf-6tf3ZxqUoiJJlZ39XFMlN9oY1xKc1bANUwdAP9_zA

# Модифицируем deployment.yaml. Добавляем еще один init-container и добавляем в него токен 
- name: metrics-init-container
        image: busybox:stable
        command:
        - wget
        - "--no-check-certificate"
        - "--header"
        - "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjRreElEdWo4SmROVVpkaXJzVzRZZFlrNnQ2eV92YjlEV0ljTkwtcjF6WUEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzE4NjE4MzQ1LCJpYXQiOjE3MTg2MTQ3NDUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJob21ld29yayIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJtb25pdG9yaW5nIiwidWlkIjoiYTVjYTQ2ZGYtOTRhZC00YThmLWI4MmItZDhmZDVhOWU4NGQ1In19LCJuYmYiOjE3MTg2MTQ3NDUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpob21ld29yazptb25pdG9yaW5nIn0.XOA1V926NzOKCHsCVGo6t340LaM29PviQ3P4Dp8y2FXy_wA28yiomFwttJcwc1ujgCLerrQksLf0qlJaWUDgpGn4-BUcFMgEFDXUf7ubHXhbFZvFn4JB0u-7rTup7I9B_Tn2YRvB0BLza-SJ-yihGY2Syy2_eY7GpsA8mf3RsFfzgIX_L07xVDU6jppE_ME3iXQtOM_9dQe3Z3Y-PzLAdCndNUnKzXrzSOd2KwMLHAApJpayN__MUpMrGqqSsh-DDqDgnuvjnPrJHj-J5GtpGqkAB5kfq6Pb5qNIX8kpUKyf-6tf3ZxqUoiJJlZ39XFMlN9oY1xKc1bANUwdAP9_zA"
        - "-O"
        - "/init/metrics.html"
        - https://10.96.0.1/metrics

# Применяем манифест deployment.yaml
nickolayfilippov@MacBook kubernetes-security % kubectl apply -f deployment.yaml
deployment.apps/controllers-pod configured

# Проверяем что metrics.html отображается корректно
![alt text](image.png)

