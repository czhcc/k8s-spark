# k8s-spark

以下使用的Spark版本为3.3.2。

## 一、生成spark镜像文件
一种方式是通过Spark源码中的命令
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag build
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag push
其中<repo>是Dockerfile文件，一般是在/kubernetes/dockerfiles/spark目录下。用这种方式的好处是，可以在镜像中加入自己特定的依赖或环境信息。

如果只是简单测试，也可以直接从网上下载镜像：
docker pull apache/spark

完成之后，可以通过命令
docker images
列出安装好的镜像。

## 二、创建namespace
为了不与其它k8s资源冲突，我看先创建一个namespace。
kubectl create -f ./spark-czh-namespace.yaml

创建完成后可以通过命令
kubectl get namespaces
列出创建的namespace。

## 三、创建PV和PVC，用来访问外部任务的JAR文件

## 四、创建一个serviceaccount

## 五、运行Spark例子

## 六、读写外部HDFS文件
