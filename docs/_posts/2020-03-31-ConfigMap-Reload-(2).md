
Stakater Reloader 오픈소스를 활용하여, ConfigMap or Secret 변경을 감지하고 Deployment 를 재기동 한다.

## Stakater Reloader
[Stakater Reloader](https://github.com/stakater/Reloader) 는 ...

### 1. Stakater Reloader 설치
```bash
$ helm install stakater/reloader --namespace kube-system
```
위와 같이 기본 설치하는 경우, 모든 Namspace 의 ConfigMap, Secret 을 감지한다.
아래 옵션 추가하여 설치하면, 특정 Namespace 만 감지 대상이 된다.
```
--set reloader.watchGlobally=false --namespace my-namespace
```

### 2. Kubernetes 리소스 설정

#### 2.1. deployment.yaml

Stakater Reloader 는 annoations 을 통해 감지, 재시작 대상을 식별한다.
아래는 현재 프로젝트의 설정이다.
```yaml
kind: Deployment
metadata:
  annotations:
    configmap.reloader.stakater.com/reload: "console-boot-template-cm"
  name: console-boot-template
  namespace: console
```

Stakater Reloader 가 지원하는 annotations 과 대상 리소스는 아래와 같다.
* `DeploymentConfig`
* `Deployment`
* `Daemonset`
* `Statefulset`

| Annotation | Type | Format |
|--|--|--|
| reloader.stakater.com/auto | string | "true" or "false" |
| configmap.reloader.stakater.com/auto | string | configmap1,configmap2,... |
| secret.reloader.stakater.com/reload | string | secret1,secret2,... |

#### 2.2. role.yaml   
namespace의 default Role에 configmaps를 get, list, watch 할 수 있는 권한을 부여한다 (why?)

```yaml
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
