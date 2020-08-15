----
Print the names of all deployments in the admin2406 namespace in the following format:
DEPLOYMENT CONTAINER_IMAGE READY_REPLICAS NAMESPACE
<deployment name> <container image used> <ready replica count> <Namespace>
. The data should be sorted by the increasing order of the deployment name.


Example:
DEPLOYMENT CONTAINER_IMAGE READY_REPLICAS NAMESPACE
deploy0 nginx:alpine 1 admin2406
Write the result to the file /opt/admin2406_data.

Hint: Make use of -o custom-columns and --sort-by to print the data in the required format.

```
$ kubectl get deployments -n admin2406 --sort-by=.metadata.name  \
      -o=custom-columns='DEPLOYMENT:metadata.name,CONTAINER_IMAGE:spec.template.spec.containers[0].image,READY_REPLICAS:status.readyReplicas,NAMESPACE:metadata.namespace' 
```

----
A kubeconfig file called admin.kubeconfig has been created in /root. There is something wrong with the configuration. Troubleshoot and fix it.

----
Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Next upgrade the deployment to version 1.17 using rolling update. Make sure that the version upgrade is recorded in the resource annotation.

```
$ kubectl create deployment nginx-deploy --image=nginx:1.16 --dry-run -o yaml > nginx-deploy.yaml
```

----
A new deployment called alpha-mysql has been deployed in the alpha namespace. However, the pods are not running. Troubleshoot and fix the issue. The deployment should make use of the persistent volume alpha-pv to be mounted at /var/lib/mysql and should use the environment variable MYSQL_ALLOW_EMPTY_PASSWORD=1 to make use of an empty root password.  
Important: Do not alter the persistent volume.

```
$ kubectl get deploy -n alpha

NAME          READY   UP-TO-DATE   AVAILABLE   AGE
alpha-mysql   0/1     1            0           12m
```


