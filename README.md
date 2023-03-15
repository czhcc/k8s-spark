# k8s-spark

以下使用的Spark版本为3.3.2。

## 一、生成spark镜像文件
一种方式是通过Spark源码中的命令 <br/>
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag build <br/>
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag push <br/>
其中<repo>是Dockerfile文件，一般是在/kubernetes/dockerfiles/spark目录下。用这种方式的好处是，可以在镜像中加入自己特定的依赖或环境信息。 <br/>
<br/>
如果只是简单测试，也可以直接从网上下载镜像： <br/>
docker pull apache/spark <br/>
<br/>
完成之后，可以通过命令 <br/>
docker images <br/>
列出安装好的镜像。 <br/>
<br/>
## 二、创建namespace
为了不与其它k8s资源冲突，我看先创建一个namespace。
kubectl create -f ./spark-czh-namespace.yaml

创建完成后可以通过命令
kubectl get namespaces
列出创建的namespace。

## 三、创建PV和PVC，用来访问外部任务的JAR文件
  由于要运行的任务JAR会经常修改，一般是放在镜像文件的外部。在生产环境可能是在HDFS或S3文件系统上。
  这里由于是在Win10的DockerDesktop下运行，就使用本南的目录。
  
  kubectl create -f spark-pv.yaml
  创建PV的具体参数，可以参考k8s的文档，留意文件中指定了前面创建的namespace名称。而里面的path路径，是DockerDesktop中的容器访问本机时使用的路径。
  
  kubectl create -f spark-pvc.yaml
  这样就把PVC也创建好了，同样是在指定的namespace下。

## 四、创建一个serviceaccount
  创建这个资源，是为了解决Spark任务提交时调用k8s接口时的权限问题，如果没有这个服务，可能会出现
  Forbidden!Configured service account doesn't have access. Service account may have been revoked.
  的异常信息。
  
  kubectl create -f spark-ServiceAccount.yaml
  文件中的具体参数，可以参考k8s的文档。

## 五、运行Spark例子  
  可以先使用镜像包里自带的JAR文件，运行一个Spark的例子。
  spark-submit.cmd 
    --master k8s://https://127.0.0.1:6443 
    --deploy-mode cluster 
    --name spark-pi 
    --class org.apache.spark.examples.SparkPi 
    --conf spark.executor.instances=1  
    --conf spark.kubernetes.container.image=apache/spark 
    --conf spark.kubernetes.namespace=spark-czh 
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=sparkuser1 
    --conf spark.kubernetes.authenticate.executor.serviceAccountName=sparkuser1 
    local:///opt/spark/examples/jars/spark-examples_2.12-3.3.2.jar
  可以看到在命令中使用了镜像名称，定义的namespace，还有serviceaccount名。local路径是运行时容器中Spark所在的路径。
  
  如果任务的JAR放在外部文件系统，则可以通过命令：
  spark-submit.cmd --master k8s://https://127.0.0.1:6443 
  --deploy-mode cluster 
  --name spark-pi 
  --class org.apache.spark.examples.SparkPi 
  --conf spark.executor.instances=1 
  --conf spark.kubernetes.container.image=apache/spark 
  --conf spark.kubernetes.namespace=spark-czh 
  --conf spark.kubernetes.driver.pod.name=my-czh-spark1 
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.options.claimName=spark-pvc-workdata0 
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.mount.path=/opt/spark/work-dir 
  --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.mount.readOnly=false 
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=sparkuser1 
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.options.claimName=spark-pvc-workdata0 
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.mount.path=/opt/spark/work-dir 
  --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.mount.readOnly=false 
  --conf spark.kubernetes.authenticate.executor.serviceAccountName=sparkuser1 
  local:///opt/spark/work-dir/jars/spark-examples_2.12-3.3.0.jar
  命令中利用了PVC，将外部存储mount到了容器中的路径/opt/spark/work-dir上，这样在local中，就可以使用外部的JAR文件了。

## 六、读写外部HDFS文件
  在Spark中执行任务时，一般会读写外部集群上的数据进行计算。这里以访问外部HDFS存储为例。
  由于要对Spark运行时的Pod配置外部hosts映射，建议使用Pod模板方式提交Spark任务。
  首先可以参考https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/，看一下Pod的hosts配置方式。
  然后我们创建一个Pod模板文件，配置时hosts信息。文件内容可以参考spark-run.yaml
  执行以下命令：
  
  spark-submit.cmd --master k8s://https://127.0.0.1:6443 
  --deploy-mode cluster 
  --name spark-pi 
  --class org.apache.spark.examples.MyHdfs1 
  --conf spark.executor.instances=1 
  --conf spark.kubernetes.driver.podTemplateFile=file:///D:/opentools/docker/spark/spark-run.yaml 
  --conf spark.kubernetes.executor.podTemplateFile=file:///D:/opentools/docker/spark/spark-run.yaml 
  --conf spark.kubernetes.driver.container.image=apache/spark 
  --conf spark.kubernetes.executor.container.image=apache/spark 
  --conf spark.kubernetes.namespace=spark-czh 
  local:///opt/spark/work-dir/jars/my_job.jar
  由于我是在Windows环境上运行，所以那个模板文件路径是Windows的写法。其它还有要注意的是，namespace和image要在命令行中指定，写在pod文件中无效。
  在Windows环境下命令中指定的yaml文件，要用file的方式。
  如果使用带有依赖包的JAR，Pod模板文件中配置外部目录的readOnly要设置为false。
