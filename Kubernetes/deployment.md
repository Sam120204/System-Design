# Deployment

官方推荐不要直接使用ReplicaSet，用Deployments。Deployment可以通过selector来匹配labels字段，过滤出它所关心的被控制对象。

Deployment不直接管理Pod对象，而是由Deployment管理ReplicaSet，再由ReplicaSet负责管理Pod对象。


Deployment是一个Kubernetes对象，在层次结构中更高。
Deployment为我们提供了使用滚动更新无缝升级底层实例的能力，e.g.撤销变化，暂停，并根据需要恢复变化。

创建一个Deployment定义文件。Deployment定义文件的内容与ReplicaSet定义文件完全相似。(在这里ReplicaSet和Deployment之间还没有太大的区别，除了Deployment创建了一个新的Kubernetes对象，叫做deployment)

```bash
kubectl create deployment my-nginx --image nginx
# deployment.apps/my-nginx created

kubectl get all
```

`vim myapp-deploy.yml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels: 
    app: myapp
    type: myservice
spec:
  selector: 
    matchLabels:
      type: myservice
  replicas: 3
  template:
    metadata:
      name: myapp-pod
      labels: 
        app: myapp
        type: myservice
    spec:
      containers:
      - name: nginx
        image: nginx
```

创建deployment
```bash
kubectl create -f myapp-deploy.yml

kubectl get deployments

kubectl get replicaset

kubectl get pods
```

### 版本升级 & 回滚

升级策略
+ recreate 一次性升级
+ rolling update 滚动升级

```bash
# kubectl create -f myapp-deploy.yml

kubectl delete deployment myapp-deployment

kubectl create -f myapp-deploy.yml && \
kubectl rollout status deployment/myapp-deployment
# deployment.apps/myapp-deployment created
# Waiting for deployment "myapp-deployment" rollout to finish: 0 of 3 updated replicas are available...
# Waiting for deployment "myapp-deployment" rollout to finish: 1 of 3 updated replicas are available...
# Waiting for deployment "myapp-deployment" rollout to finish: 2 of 3 updated replicas are available...
# deployment "myapp-deployment" successfully rolled out

kubectl rollout history deployment/myapp-deployment
# deployment.apps/myapp-deployment 
# REVISION  CHANGE-CAUSE
# 1         <none>
```
可以看见所有的pod被挨个启动，change cause为空。我们重新执行这次部署
```bash
kubectl delete deployment myapp-deployment

kubectl get pods

kubectl create -f myapp-deploy.yml --record 

kubectl rollout status deployment/myapp-deployment

kubectl describe deployment myapp-deployment
# 在annotation中出现

kubectl edit deployment myapp-deployment --record
# 修改使用nginx：alpine

kubectl rollout status deployment/myapp-deployment
# Waiting for deployment "myapp-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "myapp-deployment" rollout to finish: 1 old replicas are pending termination...
# Waiting for deployment "myapp-deployment" rollout to finish: 1 old replicas are pending termination...

kubectl describe deployment myapp-deployment
# 注意annotation 和 events

kubectl set image deployment myapp-deployment nginx=nginx:perl --record

kubectl rollout status deployment/myapp-deployment

kubectl rollout history deployment/myapp-deployment

kubectl describe deployment myapp-deployment | grep nginx

kubectl rollout undo deployment/myapp-deployment

kubectl set image deployment myapp-deployment nginx=nginx:3 --record

kubectl rollout status deployment/myapp-deployment


kubectl get deployment myapp-deployment
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# myapp-deployment   3/3     1            3           18m

kubectl get pods
# NAME                                READY   STATUS             RESTARTS   AGE
# myapp-deployment-56b894845f-7pm9v   1/1     Running            0          4m25s
# myapp-deployment-56b894845f-rs699   1/1     Running            0          4m18s
# myapp-deployment-56b894845f-svjs5   1/1     Running            0          4m22s
# myapp-deployment-67c8cfbf79-p4j6j   0/1     ImagePullBackOff   0          77s

kubectl rollout undo deployment/myapp-deployment

kubectl get pods
```



`vim myapp-deploy.yml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels: 
    app: myapp
    type: myservice
spec:
  selector: 
    matchLabels:
      type: myservice
  replicas: 3
  template:
    metadata:
      name: myapp-pod
      labels: 
        app: myapp
        type: myservice
    spec:
      containers:
      - name: nginx
        image: nginx
```

## api资源 & 缩写形式
```bash
kubectl api-resources
# NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
# bindings                                       v1                                     true         Binding
# componentstatuses                 cs           v1                                     false        ComponentStatus
# configmaps                        cm           v1                                     true         ConfigMap
# endpoints                         ep           v1                                     true         Endpoints
# events                            ev           v1                                     true         Event
# limitranges                       limits       v1                                     true         LimitRange
# namespaces                        ns           v1                                     false        Namespace
# nodes                             no           v1                                     false        Node
# persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
# persistentvolumes                 pv           v1                                     false        PersistentVolume
# pods                              po           v1                                     true         Pod
# podtemplates                                   v1                                     true         PodTemplate
# replicationcontrollers            rc           v1                                     true         ReplicationController
# resourcequotas                    quota        v1                                     true         ResourceQuota
# secrets                                        v1                                     true         Secret
# serviceaccounts                   sa           v1                                     true         ServiceAccount
# services                          svc          v1                                     true         Service
# mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
# validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
# customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
# apiservices                                    apiregistration.k8s.io/v1              false        APIService
# controllerrevisions                            apps/v1                                true         ControllerRevision
# daemonsets                        ds           apps/v1                                true         DaemonSet
# deployments                       deploy       apps/v1                                true         Deployment
# replicasets                       rs           apps/v1                                true         ReplicaSet
# statefulsets                      sts          apps/v1                                true         StatefulSet
# tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
# localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
# selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
# selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
# subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
# horizontalpodautoscalers          hpa          autoscaling/v1                         true         HorizontalPodAutoscaler
# cronjobs                          cj           batch/v1beta1                          true         CronJob
# jobs                                           batch/v1                               true         Job
# certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
# leases                                         coordination.k8s.io/v1                 true         Lease
# endpointslices                                 discovery.k8s.io/v1beta1               true         EndpointSlice
# events                            ev           events.k8s.io/v1                       true         Event
# ingresses                         ing          extensions/v1beta1                     true         Ingress
# flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta1   false        FlowSchema
# prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta1   false        PriorityLevelConfiguration
# ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
# ingresses                         ing          networking.k8s.io/v1                   true         Ingress
# networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
# runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
# poddisruptionbudgets              pdb          policy/v1beta1                         true         PodDisruptionBudget
# podsecuritypolicies               psp          policy/v1beta1                         false        PodSecurityPolicy
# clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
# clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
# rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
# roles                                          rbac.authorization.k8s.io/v1           true         Role
# priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
# csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
# csinodes                                       storage.k8s.io/v1                      false        CSINode
# storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
# volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```
