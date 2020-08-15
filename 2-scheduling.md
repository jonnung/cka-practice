## Scheduling
기본적으로 POD는 NodeName 필드를 갖고 있지만 세팅된 상태는 아니다.  
보통 POD 매니페이스트에 명시하지 않고, 쿠버네티스에서 자동으로 추가한다.  
쿠버네티스는 POD의 NodeName 속성을 직접 변경하는 것을 허용하지 않으며, Binding 오브젝트를 API 호출을 통해 생성해서 스케줄링을 모방할 수 있다.  

참고: [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)


### kube-scheduler 가 실행되고 있지 않을 때 POD를 직접 스케줄링 하기
- kube-scheduler가 실행되고 있지 않으면, POD의 `nodeSelector`를 쓸 수 없다. `nodeName` 속성으로 직접 노드 이름을 지정한다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

## labels & annotations
labels과 selector는 오브젝트를 선택하고 그룹화 하기 위해 사용된다.  
annotations는 상세 정보를 표현하는 목적으로 사용된다.  

▶︎ `env`라벨이 `prod`인 모든 오브젝트 찾기
```shell
$ kubectl get all -lenv=prod
```

▶︎ 여러 라벨로 POD 찾기
```shell
$ kubectl get po -lenv=prod,bu=finance,tier=frontend
```

## Taint & Toleration
Taints와 Toleration은 POD가 스케줄링 되는 것을 제한하는 목적으로 사용한다.  
Taints는 노드에 설정하고, Toleration은 POD에 설정한다.  

```shell
kubectl taint nodes node-name key=value:taint-effect

# Example
kubectl taint nodes node1 app=blue:NoSchedule
```

- Taint Effect : 이 Taint를 Tolerate하지 않는 POD에는 어떤 일이 벌어지는가?
	- `NoSchedule` : POD는 이 노드에 스케줄링 되지 않을 것이다. 
	- `PreferNoSchedule` :  시스템은 이 노드에 POD를 스케줄링을 하지 않으려고 노력하지만 반드시 보장하는 것은 아니다. 
	- `NoExecute` : 이 노드에 POD를 스케줄링하지 않으며, 이미 실행중인 POD가 있는 경우 Evict 된다. 

Taints와 Toleration은 POD가 어떤 노드로 스케줄링 될 지 결정하기 위해 사용하는 게 아니다.  
노드가 자신에게 부여된 Taints를 Toleration하는 POD만을 허용하는 것이다.  

참고: [테인트(Taints)와 톨러레이션(Tolerations) | Kubernetes](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/)  

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

