## Spring Cloud Gateway 구성하기
SCG (Spring Cloud Gateway)는 Engine역할을 하는 Operator가 설치가 되고, Gateway별로 인스턴스를 만들 수 있습니다.

SCG는 3개의 구성요소로 되어 있는데, 실제 Routing설정은 RouteConfig에 들어가 있고, Mapping은 Gateway 인스턴스와의 연결 설정이 들어 있습니다.

Gateway - Mapping - RouteConfig

전체적인 아키텍처 구성은 아래와 같습니다.

![](images/scg_architecture.png)

### 1. Gateway 생성하기

Gateway 를 생성하기 위해 아래의 내용으로 파일을 하나 생성합니다.
spec이하의 내용은 테스트 용도로 리소스를 적게 사용하기 위해 설정한 것으로 실제 운영을 위해서는 충분한 메모리와 cpu를 할당하는 것을 권고합니다. 아무것도 설정하지 않은 default 값의 경우에는 (요청: 1 cpu/1GB, 최대: 2 cpu/2GB) 로 설정됩니다.

hello-gateway.yaml 
```
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGateway
metadata:
  name: hello-gateway
spec:
  resources:
    requests:    
      cpu: "10m"
      memory: "500Mi"
    limits:      
      cpu: "500m"
      memory: "1000Mi"
```
생성한 파일을 실행합니다.
```
kubectl apply -f hello-gateway.yaml
```

### 2. routing config 설정하기

route-config.yaml 을 아래의 내용으로 생성합니다.
routing 설정이 들어가는 가장 핵심적인 부분으로 CRD(Custom Resource)로 관리되므로, 설정을 변경하면 동적으로 변경사항이 반영됩니다.

아래의 설정은 /hello/로 요청을 하게 되면 기존에 생성한 내부 서비스인 helloworld:8080 이름을 직접 넣어서 routing을 해주게 됩니다.
외부 URL로도 연동이 가능하며 /naver/ 로 호출한 경우 naver.com 으로 routing이 되게 됩니다.

```
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGatewayRouteConfig
metadata:
  name: hello-gateway-routes
spec:
  routes:
    - uri: http://helloworld:8080
      predicates:
        - Path=/hello/**
      filters:
        - StripPrefix=1
    - uri: https://naver.com
      predicates:
        - Path=/naver/**
      filters:
        - StripPrefix=1
```

route 설정을 적용합니다.
```
kubectl apply -f route-config.yaml
```

### 3. mapping 설정하기
이제 위에서 만든 route-config와 gateway를 연결하는 mapping설정을 하면 됩니다.
아래의 내용으로 mapping.yaml을 생성합니다.

```
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGatewayMapping
metadata:
  name: hello-gateway-mapping
spec:
  gatewayRef:
    name: hello-gateway
  routeConfigRef:
    name: hello-gateway-route
```

mapping 파일을 적용합니다.
```
kubectl apply -f mapping.yaml
```

### 4. Ingress 설정하기
기본적으로 Gateway는 ClusterIP로 동작하기 때문에 Ingress를 별도로 설정을 해주어야 합니다.
TKG에서는 Contour를 기본적으로 사용하므로 Contour를 사용하는 것을 바탕으로 합니다.

ingress.yaml 설정을 아래와 같이 합니다.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-gateway-ingress
  annotations:
    kubernetes.io/ingress.class: contour
spec:
  rules:
    - host: hello-gateway.tanzukorea.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-gateway
                port:
                  number: 80
```

여기에서 host이름은 TKG에서 사용하는 도메인이 있는 경우에는 해당 도메인을 넣으시면 됩니다. Test로 하는 경우에는 가상으로 임의의 주소를 입력을 하고, client의 pc의 /etc/hosts 파일에 URL주소와 ingress로 생성되는 외부ip를 넣으시면 됩니다.

```
kubectl apply -f ingress.yaml
```

### 5. 서비스 확인하기
생성된 ingress의 ip를 확인합니다.
```
$ kubectl get ingress
NAME                    CLASS    HOSTS                          ADDRESS       PORTS   AGE
hello-gateway-ingress   <none>   hello-gateway.tanzukorea.com   10.33.xx.xx   80      36m
```

로컬 pc의 /etc/hosts 파일에 다음을 추가합니다.
10.33.xx.xx hello-gateway.tanzukorea.com 

브라우저에서 http://hello-gateway.tanzukorea.com/hello/ 와 http://hello-gateway.tanzukorea.com/naver/ 를 호출해서 각각 다르게 routing이 되는지 확인합니다.

