### k8s整体环境的建立见README.md文件。

#### 1.镜像的建立
由于pyspark的python环境需要第三方依赖，例如访问postgresql的包sqlalchemy等，所以我将标准的apache/spark-py镜像拉下来后，进入容器，用pip将需要的依赖包安装上，重新生成了一个镜像czh_pyspark_dev_image:v3.5.1。 <br/>
进入apache/spark-py容器时缺省不是root用户，但执行pip时需要root权限，所以建立容器时指定了root用户名：-u root <br/>
<br/>
#### 2.提交任务
提交任务和前面说的java的任务基本一样：
```
spark-submit.cmd --master k8s://https://127.0.0.1:6443 
 --deploy-mode cluster 
 --name spark-pi 
 --conf spark.driver.extraClassPath=local:///mnt/jars/postgresql-42.3.3.jar 
 --conf spark.executor.instances=1 
 --conf spark.kubernetes.container.image=czh_pyspark_dev_image:v3.5.1 
 --conf spark.kubernetes.namespace=spark-czh 
 --conf spark.kubernetes.driver.pod.name=my-czh-spark1 
 --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.options.claimName=spark-pvc-workdata0 
 --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.mount.path=/mnt 
 --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.workdata.mount.readOnly=false 
 --conf spark.kubernetes.authenticate.driver.serviceAccountName=sparkuser1 
 --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.options.claimName=spark-pvc-workdata0 
 --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.mount.path=/mnt 
 --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.workdata.mount.readOnly=false 
 --conf spark.kubernetes.authenticate.executor.serviceAccountName=sparkuser1 
 local:///mnt/py/my_spark_db_2.py
 ```
 这里指定了要运行的python文件，另外，增加了一个外部依赖jar。
 使用的镜像是上面新建的镜像：czh_pyspark_dev_image:v3.5.1
