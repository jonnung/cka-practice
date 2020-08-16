## OS Upgrades
노드에 문제가 생겨서 해당 노드에 실행 중인 POD는 5분 안에(kube-controller-manger의 `--pod-eviction-timeout` 옵션에 따름) 정상화 되지 않을 경우 다른 노드에 스케줄링 된다.  
안전하게 노드를 작업하기 위해서 `drain node`를 사용한다. 이 명령은 gracefully 하게 POD를 종료시키고 다른 노드에 POD를 재생성하게 한다.  
그리고 해당 노드는 스케줄링 되지 않게 마킹된다.  (cordoned 상태)  
노드 작업이 끝난 후 다시 클러스터의 워커 노드로 투입하기 위해 `uncordon` 명령을 사용한다. 하지만 예전에 이 노드에서 다른 노드로 재생성된 POD가 다시 자동으로 돌아오는 것은 아니다.   
`drain` 명령어와 `uncordon` 명령어 외 `cordon`명령어는 스케줄링 하지 않는 다는 의미의 표시 목적으로 사용된다.   
단, `drain`과 다른 점은 이미 실행중인 POD를 종료 시키거나 다른 노드로 이주시키지 않는다. 단지 새로운 POD가 더이상 스케줄링 되지 않도록 처리한다.   

```shell
kubectl drain node01 --ignore-daemonsets
```

만약 어떤 노드에 ReplicaSets의 일부가 아닌 단독으로 실행되는 POD가 있는데 해당 노드를 그냥 Drain하게 되면 그 POD는 자동으로 Evicted 되지 않는다.  
그래서 그 POD를 강제로 제거하기 위해 `--force` 옵션을 사용할 수 있다. 하지만 그러면 그 POD는 다른 노드에 스케줄링 되지 않고 영원히 사라지게 된다.  

## Cluster upgrade process
Kube API 서버는 컨트롤 플레인의 핵심 구성요소이며, 다른 구성 요소보다 Kube API 서버 버전이 높아야 한다.  
Cotroller-manager와 Scheduler는 Kube API 버전보다 1개까지 낮아도 괜찮고, kubelet과 Kube Proxy는 2개 버전까지 낮아도 괜찮다.  
kubectl 은 Kube API 버전보다 1개 높거나 같아야 한다.   

`kube-apiserver` 버전이 `v1.10`일 때...  
`Controller-manager`, `kube-scheduler` 버전은 `v1.9` 또는 `v1.10` 가능하고, `kubelet`, `kube-proxy`는 `v1.8`, `v1.9` 또는 `v1.10`으로 가능하다. 

쿠버네티스 클러스터를 업그레이드 해야 할 시점은 언제인가?   
쿠버네티스 그룹은 최근 minor 버전 3개까지 공식 지원하기 때문에 현재 사용중인 쿠버네티스 버전보다 3단계 높은 릴리즈가 출시된 경우에 업그레이드를 고려할 수 있다.   

`kubeadm`의 `upgrade plan` 명령어를 실행하면 업그레이드 하는 데 도움이 되는 정보를 알려준다.   
클러스터 버전을 업그레이드 하기 전에 `kubeadm` 버전 먼저 올려준 뒤 작업해야 한다. 만약 현재 클러스터 버전이 `v1.11`일 때 `v1.13`으로 업그레이드를 하려고 한다면 `kubeadm`버전을 `v1.12`로 업그레이드해서 1단계씩 업그레이드를 진행해야 한다.   

- Master Node upgrade
```shell
apt-get upgrade -y kubeadm=1.18.0-00
```

```shell
kubeadm upgrade apply v1.18.0
```

위 작업까지 완료한 뒤 `kubectl get nodes` 명령어로 클러스터 버전을 확인해도 그대로인 것을 확인할 수 있다. 그 이유는 아직 kube-api에 저장된 내용이 업그레이드 되지 않은 `kubelet`버전이기 때문이다.  

- Kubelet Upgrade
```shell
apt-get upgrade -y kubelet=1.18.0-00
```

```shell
systemctl restart kubelet
```

- Worker Node Upgrade
워커 노드 하나씩 업그레이드를 진행한다.   
먼저 `kubectl drain {{워커노드이름}}`을 실행해서 해당 노드에 실행되는 POD를 다른 노드로 이주시킨다.   
그 다음 `kubectl cordon {{워커노드이름}}`을 실행해서 이 워커노드에 더이상 POD 스케줄되지 않도록 마킹한다.  

```shell
kubectl drain {{워커노드이름}} --ignore-daemonsets
```

마스터 노드에서 했던 과정과 동일하게 `kubeadm`과 `kubelet`을 먼저 업그레이드 한다.   
```shell
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
```

그리고 `kubeadm`을 통해 새로운 버전을 설정(config)하고 `kubelet`을 재시작한다.   
```shell
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet
```

모두 완료된 뒤 해당 노드에 `uncordon`으로 해당 노드에 다시 스케줄링이 될 수 있도록 설정한다. 

