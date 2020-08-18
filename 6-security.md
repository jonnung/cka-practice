## Security Primitives
Kube-apiserver는 쿠버네티스의 모든 운영 요소들 중 가장 중심 역할을 하며, kubectl은 kube-api를 통해서만 클러스터에 접근할 수 있다. 즉, kube-apiserver가 가장 첫번쩨 방어선이라 할 수 있다.  
2가지 타입의 결정이 필요하다. 누가 접근할 수 있는가? 그들은 무엇을 할 수 있는가?  

- **Authentication** (Who can access?)
	- Files - Username and password
	- Files - Username and Tokens
	- Certificates
	- External Authentication providers - LDAP
	- Service Accounts
- **Authorization** (What can they do?)
	- RBAC Authorization
	- ABAC Authorization
	- Node Authrorization
	- Webhook Mode


## Authentication
쿠버네티스는 근본적으로 사용자 계정을 관리하지 않는다. File, Certificates, LDAP 같은 서드 파티 인증처럼 외부 자원을 통해 인증을 한다.   
하지만 쿠버네티스의 ServiceAccount로 관리할 수 있는 경우도 있다.   


#### Auth Mechanisms - Basic (Static Files)
사용자 ID와 비밀번호가 담긴 CSV 파일을 이용하는 방식과 사용자 ID와 Token을 이용하는 방식이 있다.  
실제 운영 환경에서 추천하지 않는 인증 방식이다.  

~user-details.csv~
```csv
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
```

```shell
kube-apiserver --authorization-mode-Node,RBAC \
  --advertise-address=172.17.0.107
  # <중략>
  --basic-auth-file=user-details.csv
```

```shell
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:password123"
```

~user-token-details.csv~
```
KpsjVjskfVjskdfi2jvSkdjfkvblsjfD8eshsfj,user10,u0010,group1
sdfVJLDJEofjwlsldfpqjtenb927slvJDds0sdD,user11,u0012,group1
Xkdjwifjws4hjdkh328sjxmfksjdflwmfDKKjew,user12,u0012,group2
```

```shell
kube-apiserver --authorization-mode-Node,RBAC \
  --advertise-address=172.17.0.107
  # <중략>
  --token-auth-file=user-token-details.csv
```

```shell
curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer KpsjVjskfVjskdfi2jvSkdjfkvblsjfD8eshsfj"
```


#### Service Account
> 이 오브젝트는  CKA 시험 범위에 포함되지 않음, CKAD에 해당됨

#### Certificate in Kubernetes
Root Certificate를 기반으로 만들어진 Client Certificate, Server Certificate가 존재하게 된다.   
쿠버네티스 클러스터의 마스터 노드와 워커 노드간 통신이나 `kubectl`사용자와 kube-api server간 통신은 모두 TLS 보안이 적용된다.   
따라서 쿠버네티스 클러스터 내부의 서비스들은 모두 Server Certificate를 사용해서 통신을 암호화하고, 클라이언트는 Client Certificate를 사용해서 자신의 신분을 증명해야 한다.  
하지만 Certificate는 인증된 기관을 통해 서명되어야 하기 때문에 Certificate Authority(CA)가 필요하다.  
쿠버네티스 클러스터를 위한 CA는 브라우저로 접속하는 웹 사이트에서 제공하는 TLS 인증서의 CA와 원리는 같다고 할 수 있지만 코모도, 시만텍, 고대디 등과 같은 인증 기관으로부터 서명을 받는 대신 자체적으로 서명한 CA를 사용한다.   
이 CA를 Root Certificate라고 하며, CA를 만들게 되면 `ca.crt`, `ca.key`을 갖게 된다.  
Certificates의 Public key는 보통 `.crt` 또는 `.pem`이라는 이름을 갖고, Private key는 `.key` 또는 `*-key.pem` 이름을 사용한다.  


##### - Server Certificates for Servers
kube-api server의 경우, 사용자가 HTTP 요청을 통해 쿠버네티스 클러스터를 관리할 수 있기 때문에 모든 통신은 보안이 되어야 한다.   
따라서 일반적인 웹 서버의 HTTPS 제공 방식과 동일하게 TLS Certificate와 Private Key가 필요하다.  
`kubeadm`으로 클러스터를 구축하면 `/etc/kuberntes/pki` 디렉터리 밑에 `apiserver.crt`와 `apiserver.key`가 생성되어 있다.   
그 밖에 etcd 서버도 동일한 목적으로 `etcdserver.crt`, `etcdserver.key`이 만들며, 마스터 노드와 워커 노드에 실행 중인 Kubelet도 Kube-api server에서 상태 관리를 위한 통신을 위한 HTTP API 서비스를 제공하기 때문에 `kubelet.crt`, `kubelet.key`를 만들게 된다.   

##### - Client Certificates for Clients
Client Certificates는 다양한 사용자(Client)가 자신의 신분을 증명하기 위해 사용하는 Certificates이다.   
`kubectl`을 사용하는 클라이언트는 Kube-api server와 통신하기 위해 `admin.crt`와 `admin.key`를 만들어야 한다.   
Kube-Scheduler도 POD를 스케줄링하기 위한 정보를 얻기 위해 Kube-api server와 통신하기 때문에 `scheduler.crt`,`scheduler.key`를 만들어야 한다. Kube-Scheduler는 관리자 클라이언트와 비슷한 클라이언트일 뿐이다.   
그밖에 Kube-Controller-Manager와 Kube-Proxy도 Kube-api server와 통신해야 하기 때문에 위와 동일하게 Key 페어를 갖고 있어야 한다.   

kube-api server는 ETCD 서버와 통신하기 때문에 ETCD 서버 입장에서 Kube-api server는 클라이언트이다.   
따라서 Kube-api server도 etcd 서버와 통신하기 위한 Certificate 키 페어를 가져야 한다.  
이미 만든 `apiserser.crt`와 `apiserver.key`를 사용해도 되고 새로운 키 페어를 만들어도 된다. (`apiserver-etcd-client.crt`, `apiserver-etcd-client.key`)  

Kube-api server는 워커 노드 상태를 모니터링하기 위해 워커 노드에서 실행 중인 Kubelet server와도 통신한다. 이 경우에도 마찬가지로 신분 인증을 위해 Certificate 키 페어가 필요하다. (`apiserver-kubelet-client.crt`, `apiserver-kubelet-client.key`)  

##### - Certificate Authority
위에서 다룬 Server Certificate와 Client Certificate에 사인을 하기 위한 Root Certificates가 존재한다.   
이 Root Certificate는 `ca.crt`와 `ca.key` 쌍으로 존재하고, ETCD에서 사용하는 Client Certificate와 Server Certificate에 사인하기 위한 `ca.crt`와 `ca.key`가 따로 존재하기도 한다.   

#### TLS Certificates
1. the server first sends a certificate signing request (CSR) to a CA.
2. the CA use its private key to sign the CSR. the signed certificate is sent back to the server the server configure is the web application with the signed certificate.
3. whenever user access the web application. the server first sends the certificate with its **public key**.
4. user's browser reads the certificate and use the CA's public key to validate and retrieve the server's public key.
5. it then generates a **symmetric key** that it wishes to use going forward for all communication.
6. the symmetric key is **encrypted** using the server as public key and sent back to the  server.
7. the server uses its private key to **decrypt** the message and retrieve the symmetric key.


**Certificates** with **public key** are named `crt` or `pem` extension.  
so that's `server.crt`, `server.pem` for server certificates or `client.crt` or `client.pem` for client certificates.  
**Private keys** are usually with extension `.key` or `-key.pem`.  



#### TLS in Kubernetes
##### - Certificate Authority
먼저 CA(Certificate Authority) 측면에서 루트 인증서(root certificate)를 만들도록 한다.  
인증서를 만들기 위한 여러가지 방법 중 `openssl`을 이용한다.  
먼저 Private key를 만들고, CSR 파일을 만든다.  
CSR에는 [CN(Common Name)](https://knowledge.digicert.com/solution/SO7239)) 필드에 정보를 입력해줘야 한다. CN 필드의 용도는 보통 웹 사이트에서 HTTPS를 제공하기 위해 사용하는 인증서에는 CN 필드는 FQDN을 명시하도록 한다. 그래야 클라이언트가 요청하는 서버와 출처가 같은지 확인할 수 있기 때문이다.   
마지막으로 인증서를 만들기 위해 `x509` 명령어를 사용한다. 처음에 만들었던 Private key를 이용해 CSR에 명시에 따라 인증서에 서명하게 된다.   
이렇게 CA는 Private key와 인증서를 갖게 되었다.  

```shell
# Private key
openssl genrsa -out ca.key 2048

# Certificate Signing Request
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Signing Certificate
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```


##### - Client Certificate : ADMIN USER
admin 사용자를 위한 인증서를 만들어본다.  
전 단계와 동일하게 Private Key를 만든 후 CSR을 만든다. 이 CSR에도 CN 필드 값을 넣어줘야 한다. 이 때 CN 필드 값은 `kube-admin`같이 사용자 이름을 명시하면 된다. 꼭 `kube-admin`일 필요는 없으며 단지 `kubectl`을 이용할 때 쿠버네티스에 인증하기 위한 사용자를 의미하고, audit log 같은 곳에 이 이름이 남게 된다.  
마지막으로 x509 명령어로 서명된 인증서를 만든다.  
 단, 이 클라이언트 인증서는 전 단계에서 만든 루트 인증서의 하위 인증서이기 때문에 Certificate Authrity의 `ca.crt`와 `ca.key`를 활용하게 된다.   

```shell
# Private key
openssl genrsa -out admin.key 2048

# Certificate Signing Request
openssl req -new -key admin.key -subj "/CN=kube-admin" -out admin.csr

# Signing Certificate
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

이로써 클러스터 내부에서 유효한 인증을 받을 수 있는 인증서가 만들어지게 된다.   
이렇게 Private Key와 인증서를 만드는 과정은 새로운 사용자(계정)을 만드는 방식과 유사하다고 볼 수 있다.   
인증서는 사용자의 ID 역할을 하고 Private Key는 비밀번호 역할을 하는 것이다.  

이렇게 만든 admin 사용자는 다른 사용자와 어떻게 구별될 수 있을까?  
이 부분은 Group을 이용해서 해결할 수 있다. 쿠버네티스 내부에는 이미 정해진 Group이 몇가지 있는데 그 중 `system:masters` 그룹이 관리자 권한을 갖고 있다. admin 사용자를 이 그룹에 속하기 위해 CSR 생성시 `O`필드를 통해 소속 그룹을 명시할 수 있다.   

```shell
# Certificate Signing Request
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
```

##### - Serverside Certificate
ETCD 서버의 Certificate도 이전 과정과 차이가 없다. ETCD 서버는 고가용성을 위해 여러 서버로 구성될 수 있는데 각 멤버들이 보안이 적용된 통신을 하기 위해 추가로 Peer Certificate를 만든다.   
그리고 ETCD 서버가 실행될 때 이 Key와 Certificate를 명시해줘야 한다.   

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.187.2.31:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://10.187.2.31:2380
    - --initial-cluster=k8s-test001=https://10.187.2.31:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.187.2.31:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://10.187.2.31:2380
    - --name=k8s-test001
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

#### View Certificate Detail
Certificate 상세 정보를 보는 명령어 

```shell
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

#### Certificate Workflow & API
새로운 클러스터 관리자가 생겼을 때 쿠버네티스 마스터 노드에 저장된 Root Certificate를 이용해서 CSR을 만든 후 새로운 관리자를 위한 admin용 Client Certificate를 만들 수 있다.   
하지만 이 작업은 마스터 노드에 직접 들어와서 해야 하는 부담과 함께 관리자가 점점 늘어나거나 만료 기간이 지났을 때 마다 해야 하는 번거로움이 있을 수 있다.   
쿠버네티스는 하위 Certificate를 요청하는 작업을 할 수 있는 API를 제공한다. 이 API를 통해서 `CertificateSigningRequest`를 보내고, 관리자가 `kubectl`을 이용해 확인한 후 승인해서 전달할 수 있다.   

이 과정은 크게 4가지 단계로 이뤄진다.  

1. Create CertificatesSigningRequest Object
2. Review Requests
3. Approve Requests
4. Share Certs to users

이전 과정에서 CSR을 만드는 것과 동일하게 새로운 사용자는 `openssl genrsa` 명령어로 Private Key를 만들고 CSR까지 만든다. 그 다음 쿠버네티스의 `CertificateSigningRequest`는 오브젝트를 정의한 yaml 형식의 매니페스트 파일을 작성한다.   
`CertificateSigningRequest` 오브젝트의 `request`항목에 CSR 텍스트를 입력하며, 평문 그대로 입력하지 않고 BASE64 인코딩한 결과를 입력하도록 한다.   

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  groups:
  - system:authenticated
  usages:
  - client auth
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
```

```shell
kubectl get csr
```

```shell
kubectl certificate approve john
```

생성된 Certificate는 yaml 형식으로 가져온 다음 `status.certificate` 속성값을 다시 BASE64으로 디코딩한 결과를 사용하면 된다.   

```shell
kubectl get csr john -o yaml
```

쿠버네티스 클러스터의 마스터 플레인 구성 요소 중 모든 Certificate 관련 작업을 수행하는 컴포넌트는 Controller Manager이다.   


## Image Security
How do you pass the credentials to the docker runtime on the worker node for that.  
We first create a secret object with the credentials in it.  
The Secret is of type Docker registry and we name it `regcred`.  
We then specify the secret inside our pod definition file under the `imagePullSecret` section when the pod created kubernetes or the kubelets on the worker node uses the credentials from the secret to pull images well.  


#### Securty Contexts
도커 컨테이너를 실행할 때 `cap-add` 파라미터로 리눅스 Capability를 추가 할 수 있다.   
쿠버네티스의 POD는 컨테이너를 캡슐화 한 것이기 때문에 동일하게 이 설정을 POD 레벨과 컨테이너 레벨에서 적용할 수 있다.   
만약 POD 레벨과 컨테이너 레벨 모두 설정된 경우 컨테이너 레벨의 설정이 우선시 된다.   

POD 매니페스트 상에서 `spec` 섹션에 `securityContext`를 추가한다.   
컨테이너 레벨에 설정하려면 `securityContext`섹션을 `containers` 아래 해당하는 컨테이너 설정으로 이동한다.   

 `capabilities` 설정은 오직 컨테이너 레벨에서만 사용할 수 있다.  


#### Network Policies
Network Policy는 POD 내부로 들어오는 Ingress 트래픽과 외부로 나가는 Egress 트래픽을 제안할 수 있는 쿠버네티스의 네임스페이스에 속하는 오브젝트이다.  
쿠버네티스 안에서 모든 네트워크 통신은 열려있지만 Network Policy가 POD에 적용되는 경우 허용되는 트래픽 설정만 통신이 가능하다.   
Network Policy는 ReplicaSet이나 Service처럼 Label을 통해 대상 POD를 지정할 수 있다.   

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
	  matchLabels:
	    role: db
	policyTypes:
	- Ingress
	ingress:
	- from:
	  - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

Network Policy는 쿠버네티스 클러스터에서 사용하는 네트워크 솔루션을 통해 동작한다.  
`Kube-router`, `Calico`, `Romana`, `Weave-net`에서는 Network Policy가 적용되며, `Flannel`은 제공하지 않는다.  
`Flannel`을 사용하고 있더라도 Network Policy 오브젝트를 생성하는 것은 가능하지만 적용되지는 않는다.  


