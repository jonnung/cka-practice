## kube-apiserver
- 쿠버네티스에서 가장 핵심이 되는 컴포넌트
- `kubectl`로 수행하는 동작은 kube-apiserver와 통신을 주고 받으면서 이뤄진다. 
- kube-apiserver는 요청을 받으면 가장 먼저 인증(Authenticating)을 수행하고, Validation을 확인한다. 그리고 ETCD 클러스터로부터 데이터를 가져온 뒤 요청한 정보를 반환한다. 
- POD가 생성되는 과정
  - kube-apiserver는 POD를 직접 노드에 할당하지 않고, ETCD 서버에 POD 오브젝트 정보를 생성하고 수정하는 역할만 한다. 그리고 사용자에게 POD가 생성되었다 라는 정보를 알려준다. 
  - 스케줄러는 지속적으로 kube-apiserver를 모니터링하고 있다가 새로운 POD가 생성되길 원한다는 정보를 알게 된다. 
- Assigned the scheduler identifies the right node to place the new POD on and communicates that back to the kube-apiserver
  - 스케줄러는 새로운 POD 생성할 올바른 노드를 알아내고, 다시 kube-apiserver에게 이 정보를 전달한다.
  - kube-apiserver는 이 정보를 ETCD 클러스터에 업데이트한다. 
  - kube-apiserver는 새로운 POD를 생성하길 원하는 워커 노드의 kubelet으로 정보를 전달한다. 
  - kubelet은 해당 노드에 새로운 POD를 만들어 새로운 애플리케이션 이미지를 배포하기 위해 컨테이너 런타임(도커)에게 명령을 전달한다.  
  - kubelet은 처리 상태를 다시 kube-apiserver에서 돌려주고, kube-apiserver는 이 데이터를 ETCD 클러스터에 저장한다.
- kube-apiserver는 요청에 대한 인증 및 검사 절차와 데이터를 ETCD 클러스터에 저장하고 가져오는 역할을 수행한다. 


## kube-controller-manager
- 컨트롤러는 시스템 내 다양한 구성 요소의 상태를 지속적으로 모니터링하고 전체 시스템을 원하는 기능 상태로 만드는 프로세스다. 
- 노드 컨트롤러는 노드들의 상태를 모니터링하고, 애플리케이션이 계속 실행될 수 있도록 필요한 조치를 수행한다. (이 작업들은 kube-apiserver를 통해 이뤄진다)
- 노드 컨트롤러는 5초 마다 상태를 체크한다. 이 옵션은 `--node-monitor-period=5s`로 지정한다. 
- 노드의 상태가 정상적이지 않은 경우 약 40초 정도 기다린 후 `unreachable` 상태로 표시된다. 이 옵션은 `--node-monitor-grace-period=40s`로 지정한다. 
- 만약 노드가 5분 안에 정상화 되지 않으면, 해당 노드에 할당된 POD는 삭제되고 다른 정상적인 노드에 다시 생성될 수 있도록 한다. 이 옵션은 `--pod-eviction-timeout=5m0s`으로 지정한다. 
- kube-controller-manager는 여러가지 컨트롤러가 하나의 프로레스로 패키징되어 실행되는 형태이다. 


## kubelet
- kubelet은 선박의 선장과 같은 역할을 한다. 
- kubelet은 워커 노드에 POD를 생성하기 위해 컨테이너 런타임에 요청을 전달한다. 
- kubelet은 POD와 컨테이너의 상태를 지속적으로 모니터링하고 그 결과를 kube-apiserver에 주기적으로 전송한다. 
- kubeadm으로 클러스터의 워커 노드를 구성할 경우 kubelet이 자동으로 설치되고 실행되는 것은 아니다. 반드시 별도로 kubelet을 다운로드하고 설치 후 실행해야 한다. 


## kube-proxy
- 쿠버네티스 클러스터의 모든 POD는 서로 다른 POD와 통신할 수 있다. 이것이 가능한 이유는 클러스터 내 POD 네트워크를 담당하는 기능이 배포되어 있기 때문이다. 
- POD 네트워크는 클러스터 내 모든 노드에 내부적인 가상 네트워크를 통해 모든 POD가 통신할 수 있도록 해준다. 
- a better way for the web application to access the database is using a **service**.
- 웹 애플리케이션 POD에서 DB가 실행되고 있는 POD에 접근하기 위한 가장 좋은 방법은 Service를 이용하는 것이다. 
- Service는 고유한 IP를 할당 받고, FQDN을 통해 접근할 수 있기 때문에 어떤 POD에서도 접근할 수 있다. Serivce는 전달 받은 요청을 자신이 담당하는 백엔드 POD로 전달하는 역할을 한다.
- Service는 어떻게 IP를 얻는 것이며, 같은 POD 네트워크에 조인할 수 있는 것인가?
  - Service는 POD 네트워크에 조인하지 않는다. 왜냐하면 Service는 실제 존재하는 것이 아니기 때문이다. 
  - POD와 같이 컨테이너로 실행되지 않으며 어떤 네트워크 인터페이스나 요청을 기다리는 프로세스가 존재하는 것이 아니다. 
  - 이것은 가상 컴포넌트 개념이며 쿠버네티스 메모리 상에 존재하는 것일뿐이다. 
- kube-proxy는 쿠버네티스 클러스터 내 모든 노드에서 실행되는 프로세스다. 
- 이것의 역할은 새로운 Service를 만들고 Serivce를 찾는 역할을 수행한다.
- 어떤 서비스가 어떤 백엔드 POD로 전달될 수 있는지 같은 네트워크 규칙을 만든다.
- 이와 같은 네트워크 규칙을 만드는 한가지 방법으로는 IPTABLES을 사용하는 것이다. 
- 모든 노드에 Service의 IP로 들어오는 트래픽에 대해 실제 POD로 보낼 수 있는 규칙을 IPTABLES에 만들어 준다.
- kubeadm은 모든 노드에 kube-proxy를 배포하며 이는 DaemonSet으로서 실행된다. 따라서 모든 클러스터의 노드마다 오직 하나의 kube-proxy POD가 실행될 수 있다.


## ReplicaSet
Replication Controller와 큰 차이점은 `selector`를 지정해서 일치하는 `labels`을 가진 POD에 대한 리플리카를 관리할 수 있는 점이다.


## Namespace
`ResourceQuota`오브젝트를 이용해 네임스페이스마다 리소스를 제한할 수 있다.

## kubectl usages
**Create an NGINX Pod**
```shell
$ kubectl run nginx --image=nginx
```

**Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)**
```shell
$ kubectl run nginx --image=nginx -o yaml --dry-run=client > nginx_pod.yaml
```


**Create a deployment**
```shell
$ kubectl create deployment nginx --image=nginx
```

**Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)**
```shell
$ kubectl create deployment nginx --image=nginx -o yaml --dry-run=client
```


**Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379**
```shell
$ kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
위 경우 자동으로 `redis` POD의 labels을 selector로 지정된다. 


```shell
$ kubectl create service clusterip redis --tcp=6379:6379 -o yaml --dry-run=client
```
이렇게 Service를 만드는 경우 POD의 labels을 selectors에 지정되지 않지만 `app=redis`라는 labels을 추정하여 selectors로 지정된다. 


**Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:**
```shell
$ kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml
# 이 명령어에서 NodePort는 적용되지 않는다. 
```
```shell
$ kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```
위 두 명령어에서 하나는 NodePort를 지정할 수 없고, 하나는 selectors를 사용할 수 없다. 
따라서 `expose` 명령어를 주로 사용하뒤 Definitions 파일을 통해 NodePort를 지정하는 것이 낫다.


## kubectl apply
왜 'last applied configurations'가 필요한가?  
Local 설정 파일과 'last applied configuration'을 비교해서 삭제된 필드가 무엇인가 확인한 뒤 Live 설정에서 해당 필드를 삭제할 수 있게 된다.   
'last applied configuration'은 Live 설정 상에 `annotaions` 밑에 `kubectl.kubenetes.io/last-applied-configuration`에 JSON 포맷으로 저장된다.  
