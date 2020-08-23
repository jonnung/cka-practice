## Networking
### Linux Networking Basics
어떤 A, B 시스템이 있다면 어떻게 네트워크로 연결될 수 있을까?  
Switch는 여러 시스템을 하나의 네트워크로 연결해주는 역할을 한다. 이 시스템에서 스위치를 통해 연결되기 위해서 반드시 인터페이스가 필요하다.   
아래 명령어를 사용해서 네트워크 인터페이스 연결 여부를 확인할 수 있다.   

```shell
ip link
```

그리고 `ip addr` 명령어를 사용하면 연결된 네트워크 인터페이스에 할당된 IP 주소를 확인할 수 있다.   

```shell
ip addr add 192.168.1.10/24 dev eth0
```

만약 192.168.1.0/24 네트워크와 192.168.2.0/24 네트워크가 존재한다면, 이 네트워크는 서로 다른 네트워크를 의미하기 때문에 직접적으로 통신할 수 없다.  
**Router**는 서로 다른 네트워크를 연결해주는 역할을 한다.   
그래서 Router에는 각 다른 네트워크를 통해 서로 IP를 하나씩 할당 받게 된다. (192.168.1.1, 192.168.2.1)  

**Gateway**는 다른 네트워크나 인터넷으로 나가기 위한 관문 역할을 한다. 시스템은 이 문을 통해 어디로 나가야할 지 결정하는 라우팅 설정을 갖고 있다.   
`route` 명령어를 통해 커널 라우팅 테이블을 확인할 수 있다.   
만약 `192.168.1.11`에서 `192.168.2.10`으로 패킷을 보내려 한다면, 다음과 같이 `ip route`명령어로 Gateway를 지정할 수 있다.   

```shell
ip route add 192.168.2.0/24 via 192.168.1.1
```

인터넷 상에 있는 네트워크(예: `172.217.194.0`)에 연결하기 위해서도 동일한 과정으로 라우팅 설정을 하면된다.  
어떤 다른 네트워크에 대해 기본 Gateway를 통해 나가게 하려면 아래와 같이 default 설정을 한다.  

```shell
ip route add default via 192.168.2.1
```

default 설정 시 `default` 항목을 `0.0.0.0`으로 지정하면 동일한 의미를 갖는다.  
Router가 여러개일 경우 각 Router마다 Gateway를 위한 IP를 설정하고, 라우팅 설정에서 목적지 네트워크 주소(예: default 또는 다른 네트워크 대역)를 지정하면 된다.  


A 호스트가 C 호스트와 통신하기 위해서 우선 B 호스트로 연결되는 Gateway 설정을 해야 한다. 

```shell
# 예시
ip route add 192.168.2.0/24 via 192.168.1.6
```

요청을 받은 C 호스트가 A호스트에게 응답을 전달하려면 위 설정과 마찬가지로 Gateway 설정이 필요하다.  
하지만 실제로 Ping이 도달하지 않는 현상이 발생한다.  
리눅스는 기본적으로 하나의 인터페이스로 들어온 패킷을 다른 인터페이스로 전달되지 않는다. 위 경우에는 B 호스트의 `eth0`로 수신된 패킷은 `eth1`을 통해 다른 곳으로 전달되지 않는다. 그 이유는 보안적인 이유 때문이다.   
만약 B 호스트에 연결된 네트워크가 Private과 Public이라면, 악의적인 누군가가 Public 네트워크의 접근을 통해 Private 네트워크로 접근할 수 있기 때문이다.  
하지만 이 경우에는 Private 네트워크간 중계를 담당하기 때문에 이러한 보안 설정을 임의로 조정할 수 있다.  
`/proc/sys/net/ipv4/ip_forward` 파일의 숫자가 기본값이 0이지만 이 값을 1로 변경하면 된다. 이 설정은 시스템 재부팅시 초기화 되므로 `/etc/sysctl.conf` 파일도 수정해줘야 한다.   

### DNS
`/etc/resolv.conf`에 네임 서버를 명시할 수 있고, `search` 항목에 탑 레벨 도메인을 명시한다.  
해당 호스트에서 어떤 호스트를 찾기 위한 질의를 할 때 `search`에 명시된 탑 레벨 도메인의 하위 서브 도메인인지 먼저 찾아보게 된다.  

### Network Namespace in Linux
리눅스 커널 네임스페이스는 프로세스와 네트워크를 격리 시킬 수 있는 기능이다.  
네임스페이스 내에서 실행되는 프로세스는 PID 1을 가지면 root 권한으로 실행된다. 이 프로세스는 실제로 호스트 환경에서도 확인할 수 있지만 PID 1이 아닌 다른 PID를 갖고 있는 것을 확인할 수 있다.  

네임스페이스를 통해 네트워크를 구성하기 위해서는 아래 명령어를 사용한다.  

```shell
ip netns add red
```

호스트에서 네트워크 인터페이스를 확인할 수 있는 명렁어인 `ip link`를 네임스페이스의 네트워크에서도 실행할 수 있다.  

```shell
ip netns exec red ip link

# 위 명령어와 동일한 기능을 한다.
ip -n red link
```

네임스페이스 상의 네트워크는 `lo <LOOPBACK>` 인터페이스만 확인되며, 호스트의 `eth0`인터페이스는 볼 수 없다.  
따라서 네임스페이스 상의 네트워크는 인터넷에 연결될 수 없다.   
인터넷에 연결하기 위해서는 Virtual Ethernet Pair를 이용해야 한다.   
이것은 종종 Pipe와 관련되어 취급되지만 두 인터페이스를 Virtual Cable로 연결된 것처럼 생각하면 된다.   
케이블을 만들기 위해서 `link` 명령어의 `peer`를 이용해서 ......   

> TODO: 강좌를 다시 듣고 자세히 정리할 것

### Docker Network
`none`네트워크 : 어떤 네트워크와도 연결되지 않는다. 다른 컨테이너끼리 통신도 되지 않는다.  
`host` 네트워크 : 호스트 네트워크를 공유한다. 단, 여러 컨테이너가 같은 포트를 사용할 수는 없다.  
`bridge` 네트워크 : 내부 Private 네트워크를 만든 후 호스트 네트워크와 연결된다.  

> 이전 강좌를 완전히 이해한 후 다시 볼 것

### Container Network Interface
CNI는 컨테이너 런타임 환경에서 네트워킹 문제를 해결하기 위해 프로그램을 개발하는 방법을 정의하는 표준 집합이다.  

### Container Networking Interface (CNI) in Kubernetes
CNI는 컨테이너 런타임 환경에서 서로 네트워킹 문제를 해결하기 위해 프로그램을 개발하는 방법을 정의한 표준이다.  

서로 다른 노드의 POD끼리 통신을 위해서 각 노드 환경마다 Bridge 네트워크와 라우팅 테이블을 준비해야 하는 작업을 해야한다.  
이 작업들은 각 단계별로 필요한 단계를 수행하는 스크립트를 작성해서 각 컨테이너가 실행될 때마다 수동으로 실행되도록 해야 했다. 하지만 큰 서비스 환경에는 이 같은 수동 작업은 많은 리소스 낭비가 발생한다.  

그래서 쿠버네티스의 POD가 실행될 때 자동으로 이런 스크립트를 실행 시켜줄 수 있는 중간자 역할을 하는 것이 CNI이다.   
CNI는 쿠버네티스가 컨테이너를 생성하자마자 이 스크립트를 호출하는 방법을 알려주는 식이다.   

kubelet은 컨테이너 생성을 담당하는 역할을 수행한다.  
kubelet이 컨테이너를 생성될 때마다 kubelet 프로세스 실행시 전달 받은 CNI 설정 정보(`--network-plugin=cni`)를 통해 실행 가능한 CNI 플러그인 파일(`--cni-bin-dir`)과 설정 파일(`--cni-conf-dir`)을 찾는다 그리고 스크립트의 `add`명령어를 수행하게 되는 것이다.  


### CNI Weave Works
라우팅 테이블은 많은 엔트리를 포함할 수 없기 때문에 다른 해결책이 필요하다.   
CNI 플러그인이 클러스터에 배포되면 모든 노드에 에이전트처럼 배포된다. 이 에이전트들은 서로 통신해서 노드와 네트워크, POD에 대한 정보를 교환한다.  
네트워크 패킷이 어떤 POD에서 다른 노드로 전달될 때 Weave가 이 패킷을 가로챈 뒤 목적지 POD가 존재하는 노드의 네트워크 주소 정보를 포함 시켜 다시 캡슐화 해서 해당 노드로 전달한다. 그러면 해당 노드의 Weave가 다시 이 패킷을 받아 캡슐화된 정보를 확인해서 실제 목적지 POD에서 최종 전달하게 된다.  

### IPAM (IP Address Management)
IP를 어떻게 할당할지 이 정보는 어디에 저장하고 중복되지 않은 IP를 누구에게 할당할 지를 역할을 수행한다.  

**CNI Plugin Responsibilities**
- Must support argument ADD/DEL/CHECK
- Must support parameters container id, network namespace, etc...
- Must manage IP Address assignment to PODs
- Must Return results in a specific format

### Service Networking
Service가 생성되면 어떤 POD가 있는 노드이건 관계없이 클러스터의 모든 부분에서 액세스 할 수 있다. 그리고 POD와 달리 특정 노드에서 고정되어 실행되는 것이 아니다. 하지만 클러스터 내부에서만 접근할 수 있는 특징이 있다. 이같은 특징을 가진 형태를 보통 ClusterIP 타입의 Service이 된다.  

그렇다면 Service는 어떻게 IP 주소를 할당 받고 클러스터 내 모든 노드에서 접근할 수 있는 것일까?  
그리고 외부 사용자가 각 노드의 특정 포트를 통해 Service의 접근할 수 있는 방법은 무엇인가?  

각 노드에는 **kube-proxy** 라는 컴포넌트가 실행되고 있다. kube-proxy는 kube-apiserver를 통해 클러스터의 변경 사항을 지켜보고 있고, 주기적으로 새로운 Service가 생성되었는지 확인한다.   

Service는 POD와 달리 각 노드에 할당되거나 생성되지 않는다.  
Service는 클러스터 전역에 걸쳐있는 컨셉이며, 클러스터 내 모든 노드에서 존재한다고 볼 수 있다.   
하지만 사실 Service는 실제 존재는 것이 아니다. 이것은 IP를 갖지만 요청을 기다리는 어떤 서비스나 서버가 아니다. 단지 가상의 오브젝트일 뿐이다.  

쿠버네티스에서 Service를 만들면 미리 정해진 범위 내에서 IP 주소를 할당받는다.  
kube-proxy 컴포넌트는 모든 노드에서 실행되고 있고, 그 Service의 IP 주소 가져와서 노드의 포워딩 규칙을 만든다. 그래서 이 Service의 IP로 트래픽이 들어오게 될 때 다시 적절한 POD의 IP로 전달될 수 있는 것이다.  
따라서 클러스터 내 어떤 POD에서든 Service의 IP를 통해 다른 POD로 접근할 수 있다.

kube-proxy는 `userspace`, `iptables`, `ipvs` 3가지 프록시 모드를 지원한다.  기본은 `iptables`다.  

```shell
kube-proxy --proxy-mode [userspace | iptables | ipvs]
```

`kubectl get service`를 통해 나오는 결과에 표시된 `CLUSTER-IP` 대역은 kube-api-server를 실행할 때 전달한 `--service-cluster-ip-range`파라미터의 값이다.  기본값은 `10.0.0.0/24`다.  

kube-proxy는 rule을 iptables에 등록한다.   

```shell
iptables -L -t net | grep db-service
```


### DNS
쿠버네티스 클러스터에 기본적으로 `kube-dns`가 있으며, Service와 POD의 목적지 IP에 대한 FQDN을 관리하고 있다.  
Service의 경우 `{Service Name}.{Namespace}.svc.cluster.local`형식의 도메인을 가지며, POD는 `{POD IP의 점을 대시로 변경}.{Namespace}.svc.cluster.local` 형태다.  

### CoreDNS in Kubernetes
모든 POD는 생성될 때 DNS 서버의 IP가 명시된 `/etc/resolv.conf`가 추가된다.  
쿠버네티스 버전 1.12부터 CoreDNS를 추천하고 있다.  
CoreDNS는  `kube-system` 네임스페이스에 Deployment(POD)로 배포된다.  
CoreDNS 설정파일은 `Corefile`에 정의하고 `/etc/coredns` 경로에 있다.  
`Corefile`내용 중 `kubernetes`속성이 쿠버네티스 플러그인 설정이다.  

```
.:53 {
  errors
  health
  kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    upstream
    fallthrough in-addr.arpa ip6.arpa
  }
}
```

`cluster.local`은 Top 레벨 도메인을 지정한 것이며, `pods`설정은 POD에 대한 레코드 설정을 의미한다.   
`Corefile`은 `kube-system` 네임스페이스에 Configmap 오브젝트에 저장되어 있다. 설정을 변경하려면 이 Configmap을 수정하면 된다.  

각 POD의 `/etc/resolv.conf`에는 DNS 서버의 IP가 입력되어 있는데 CoreDNS도 여러 POD로 실행되기 때문에 직접 접근하기 보다 Service를 이용해야 한다. 쿠버네티스 클러스터는 기본적으로 `kube-dns`라는 Service를 생성해준다.   
이 Service의 IP는 `kubelet`실행 설정에 지정되어 있다. (`/var/lib/kubelet/config.yaml`에서 `clusterDNS`값)  

CoreDNS로 도메인을 조회할 때 FQDN으로 조회하는 원리는 각 POD의 `/etc/resolv.conf`에 `search`옵션을 통해 가능하다.   

### Ingress in Kubernetes
**Ingress**는 사용자가 외부에서 액세스 할 수있는 단일 URL을 사용하여 애플리케이션에 액세스하게 해준다. 이 URL은 URL Path를 기반으로 클러스터 내의 다른 서비스로 라우팅하도록 구성 할 수 있다. 동시에 TLS 인증서 보안도 제공한다.  

Ingress는 쿠버네티스 클러스터에 장착된 L7 로드밸런서라고 생각해도 된다.   
Ingress를 실행하기 위해서는 외부 네트워크에서 들어올 수 있는 NodePort나 Load Balancer 타입의 Serivce가 필요하다. 이것은 단 한 번만 하면 된다.   

만약 쿠버네티스가 밖에서 Ingress 역할을 직접 만든다고 하면 Nginx, HAProxy, Traefik 같은 솔루션을 이용해야 한다.   
쿠버네티스에는 이 부분을 __Ingress Controller__라고 부르는 리소스를 생성할 수 있다. 그리고 그 설정들을 __Ingress Resource__라고 하며, POD, Deployments, Services 같이 Definition file을 사용해서 생성한다.   

~Ingress Resource가 작동하려면, 클러스터는 실행 중인 Ingress Controller가 반드시 필요하다.~  
하나의 클러스터 내에  여러 개의 인그레스 컨트롤러를 배포할 수 있다. 인그레스를 생성할 때, 클러스터 내에 둘 이상의 인그레스 컨트롤러가 존재하는 경우 어떤 인그레스 컨트롤러를 사용해야 하는지 표시해주는 적절한 `ingress.class` 어노테이션을 각각의 인그레스에 달아야 한다.  

쿠버네티스는 IngressController를 기본적으로 설정하지 않기 때문에 직접 배포해야 한다.   
Ingress Controller를 직접 배포하기 위해서 사용할 수 있는 솔루션은 GCE Load Balancer, Nginx, Contour, HAProxy, Traefik, Istio 등이 있다.   
현재 쿠버네티스 프로젝트에서 GCE와 Nginx를 지속적으로 지원하고 유지하는 상태다.  

먼저 Ingress Controller를 Deployments로 배포해야 한다.   

Ingress Resource는 쿠버네티스 Definition 파일로 생성한다.   
`Ingress`에 정의된 Backend는 POD를 직접 지정하지 않고, Service를 지정한다.   

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
```

```shell
kubectl create -f Ingress-wear.yaml

kubectl get ingress
```

