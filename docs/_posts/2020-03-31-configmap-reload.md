## Purpose
ConfigMap or Secret가 변경 될 경우, 이를 인지하여 Deployment의 Pod가 자동으로 재기동 되도록 한다.    
   
#### 사용되는 OpenSource
> - [stakater/Reloader](https://github.com/stakater/Reloader)   
   
## How to use
#### 1. stakater/Reloader 설치
```
$ helm install stakater/reloader
```
Default namespace에 설치 하는 경우, 모든 namspace의 configmap, secrets을 대상으로 동작한다.   
특정 namespace에 대해서만 동작하게 하고 싶은 경우, 특정 namespace를 지정하여 설치한다.   


#### 2. 소스 코드

- src/PropertiesConfig.java   
configmap의 data필드의 값을 (@ConfigurationProperties)prefix.Field 형태로 가져와 사용한다.   
아래 코드에서는, console-boot-template-cm Configmap의 data:application.properties: 에서 가져온 값을 사용한다.   
``` java
@Configuration
@ConfigurationProperties(prefix = "bean")
public class PropertiesConfig {

    private String data = "This is default data";
    ... setter & getter
}
```

- src/RestController.java
``` java
@org.springframework.web.bind.annotation.RestController
@RequestMapping("/test")
public class RestController {
    @Autowired
    private PropertiesConfig yamlConfig;

    @GetMapping("/data")
    public String load() {
        String data = "error";
        try {
            data = String.format(yamlConfig.getData(), "", "");
        } catch (IllegalStateException e) {
            e.printStackTrace();
        }
        return data;
    }

}
```

- resources/application.properties
```
bean.data=Message from apllication.properties
```

#### 3. k8s 리소스 생성 및 수정 
- role.yaml   
namespace의 default Role에 configmaps를 get, list, watch 할 수 있는 권한을 부여한다
``` yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: ayoung
  name: default
rules:
  - apiGroups: ["", "extensions", "apps"]
    resources: ["configmaps", "pods", "services", "endpoints", "secrets"]
    verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: default-binding
  namespace: ayoung
subjects:
  - kind: ServiceAccount
    name: default
    apiGroup: ""
roleRef:
  kind: Role
  name: default
  apiGroup: ""
```

- configmap.yaml   
data:application.properties: 필드에 소스코드에서 사용하는 데이터를 입력한다
``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: console-boot-template-cm         # CHANG IT
  namespace: ayoung
data:
  application.properties: |-
    bean.data=Testing reload! Message from configmap    # CHANG IT   
```

- deployment.yaml   
metadata:annoation:configmap.reloader.stakater.com/reload에 감지 대상 ConfigMap의 이름을 지정한다
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    configmap.reloader.stakater.com/reload: "console-boot-template-cm"     # CHANGE IT
  name: console-boot-template
  namespace: ayoung
spec:
  selector:
    matchLabels:
      app: console-boot-template

  replicas: 1
  template:
    metadata:
      labels:
        app: console-boot-template
    spec:
      containers:
        - name: console-boot-template
          image: lay126/console-boot-template:latest
          ports:
            - containerPort: 8080
          env:
            - name: env.namespace
              value: ayoung
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          configMap:
            name: console-boot-template-cm
```

- service.yaml
``` yaml
kind: Service
apiVersion: v1
metadata:
  name: console-boot-template-svc
  namespace: ayoung
spec:
  selector:
    app: console-boot-template
  ports:
    - protocol: TCP
      port: 8080
      nodePort: 30083
  type: NodePort
  ```
  
- ingress.yaml
``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: console-boot-template-ing
  namespace: ayoung
spec:
  rules:
  - host: console-boot-template.cloudzcp.io
    http:
      paths:
      - backend:
          serviceName: console-boot-template-svc
          servicePort: 8080
  tls:
  - hosts:
    - console-boot-template.cloudzcp.io
    secretName: cloudzcp-io-cert
```

## 동작
#### K8S resources
``` bash
$ kubectl get all
NAME                                           READY   STATUS    RESTARTS   AGE
pod/console-boot-template-57f86b5896-hxtbk     1/1     Running   0          16m
pod/vehement-tapir-reloader-6cfd88bbff-2sszf   1/1     Running   0          13d

NAME                                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/console-boot-template-svc   NodePort   172.21.71.209   <none>        8080:30744/TCP   131m

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/console-boot-template     1/1     1            1           131m
deployment.apps/vehement-tapir-reloader   1/1     1            1           13d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/console-boot-template-57f86b5896     1         1         1       16m
replicaset.apps/vehement-tapir-reloader-6cfd88bbff   1         1         1       13d

$ kubectl get configmap
NAME                       DATA   AGE
console-boot-template-cm   1      131m
```
#### configmap edit
``` bash
$ kubectl edit cm console-boot-template-cm
---
apiVersion: v1
data:
  application.properties: bean.data=Testing reload! Message from cm!    # CHANGE IT
kind: ConfigMap
```

#### Pod Reload
``` bash
$ kgp
NAME                                       READY   STATUS              RESTARTS   AGE
console-boot-template-57f86b5896-hxtbk     1/1     Running             0          19m
# 변경된 cm반영된 Pod 생성
console-boot-template-656575f45f-rt7n6     0/1     ContainerCreating   0          2s
vehement-tapir-reloader-6cfd88bbff-2sszf   1/1     Running             0          13d
...
# 기존 Pod 중지 됨
console-boot-template-57f86b5896-hxtbk     1/1     Terminating   0          20m
```
- Configmap 수정 전
![Alt text](./img/cm-reload-before.png "cm-reload before")
- Configmap 수정 후
![Alt text](./img/cm-reload-after.png "cm-reload after")


#### Rolling Update
Deployment replicas를 3개로 조정.
``` bash
NAME                                       READY   STATUS              RESTARTS   AGE
console-boot-template-74759dcf44-9598r     1/1     Running             0          111s
console-boot-template-74759dcf44-nh9vj     1/1     Running             0          111s
console-boot-template-74759dcf44-xlnbj     1/1     Running             0          111s
```
Configmap을 수정 한 뒤, Pod의 Rolling Update를 확인.
``` bash
$ k get pod 
console-boot-template-66686d9df7-svnr9     0/1     ContainerCreating   0          1s
console-boot-template-74759dcf44-9598r     1/1     Running             0          110s
console-boot-template-74759dcf44-nh9vj     1/1     Running             0          110s
console-boot-template-74759dcf44-xlnbj     1/1     Running             0          110s
$ k get pod 
console-boot-template-66686d9df7-pdtxv     0/1     ContainerCreating   0          1s
console-boot-template-66686d9df7-svnr9     1/1     Running             0          10s
console-boot-template-66686d9df7-zwvbl     1/1     Running             0          5s
console-boot-template-74759dcf44-nh9vj     0/1     Terminating         0          119s
console-boot-template-74759dcf44-xlnbj     1/1     Running             0          119s
$ k get pod 
console-boot-template-66686d9df7-pdtxv     1/1     Running   0          10s
console-boot-template-66686d9df7-svnr9     1/1     Running   0          19s
console-boot-template-66686d9df7-zwvbl     1/1     Running   0          14s
```
