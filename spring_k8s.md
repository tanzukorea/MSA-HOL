## Kubernetes에 배포하기

## 사전 준비사항
* TKG(Tanzu Kubernetes Grid)
* kubectl 
* TKG에 접근가능한 환경(예: bootstrap) kubeconfig 파일이 위치해야 합니다. 아래의 작업은 kubectl로 TKG에 접근가능한 상태에서 작업이 가능합니다.
Local Pc에서 TKG에 접근이 가능하고(방화벽에서 6443포트 해제), kubeconfig 파일이 Local PC에 위치해야 합니다.

### 1. k8s 배포 yaml 파일 생성하기

아래 명령어를 실행해서 k8s yaml 파일을 생성합니다.
```
kubectl create deployment helloworld --image=[Harbor주소]/helloworld:1.0 --port=8080 --dry-run=client -o=yaml >> deploy.yaml

kubectl create service loadbalancer helloworld --tcp=8080:8080 --dry-run=client -o=yaml >> service.yaml
```

deploy.yaml 이 아래와 같이 생성이 되게 됩니다.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: helloworld
  name: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helloworld
    spec:
      containers:
      - image: projects.registry.vmware.com/cnr/helloworld:1.0
        name: helloworld
        ports:
         - containerPort: 8080
        resources: {}
status: {}
```

service.yaml이 아래와 같이 생성됩니다.
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: helloworld
  name: helloworld
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: helloworld
  type: LoadBalancer
status:
  loadBalancer: {}
```

### 2. k8s에 적용하기
```
kubectl apply -f deploy.yaml
kubectl apply -f service.yaml
```

### 3. k8s 배포 확인하기
Pod와 서비스가 정상적으로 생성이 되었는지 확인합니다.
```
❯ kubectl get pods
NAME                                          READY   STATUS    RESTARTS   AGE
helloworld-68f8df955c-kf4qz                   1/1     Running   0          2m7s

❯ kubectl get services
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
helloworld                LoadBalancer   100.68.200.37    10.3.x.x      8080:32570/TCP      5m47s
```

브라우저에서 EXTERNAL-IP 의 8080 포트로 접속해서 페이지가 정상적으로 열리는지 확인합니다.
curl 명령어로도 확인하실 수 있습니다.
```
> curl http://10.3.x.x:8080
Hello World%
```
