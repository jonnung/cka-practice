## Monitoring Cluster Component
Heapster는 쿠버네티스 모니터링과 통계 기능을 위한 오리지널 프로젝트였다.  
그러나 Heapster는 현재 Deprecated된 상태이며 앞으로는 Metrics Server를 사용하는 것을 권장하고 있다.  

쿠버네티스의 각 워커 노드마다 실행되는 kubelet은 kube-apiserver를 통해 명령을 전달 받고 POD를 실행하는 역할을 담당한다.  
그리고 kubelet은 서브 컴포넌트로서 cAdvisor(Container Advisor)를 포함하고 있다.  
cAdvisor는 POD로부터 성능 지표 데이터를 받아와 Metrics Server에서 사용할 수 있는 지표로 만들고, Metrics Server는 각 노드의 kubelet API에 쿼리해서 데이터를 수집해간다.   

[https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)


