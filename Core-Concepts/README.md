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

