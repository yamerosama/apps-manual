1. 설치
python - www.python.org/downloads에서 운영체제에 맞는 package download(2021.03.28 현재 v3.9.2가 최신임)
cloud:/Users/cloud $ python -V
Python 3.9.1
Pycharm - www.jetbrains.com/pycharm/download 에서 OS에 맞는 pycharm 설치가능함. (2021.04.25 현재 v2021.1.1가 최신임)
2. Bitnami spark docker image
기본적으로 python 3.6.x 버전으로 되어 있어서 base image를 python:3.8-slim-buster로 하여 따로 이미지를 생성함.
Spark 3.1.1 임.
LD_LIBRARY_PATH에 /opt/bitnami/spark/venv/lib/python3.6/site-packages/numpy.libs 가 있는데 왜 일까?
우선 git clone 부터 하자

$ git clone https://github.com/bitnami/bitnami-docker-spark
3/debian-10/Dockerfile을 수정하여 docker build 함.

#FROM docker.io/bitnami/minideb:buster

FROM python:3.8-slim-buster
LABEL maintainer "Bitnami <containers@bitnami.com>"

ENV HOME="/" \
    OS_ARCH="amd64" \
    OS_FLAVOUR="debian-10" \
    OS_NAME="linux" \
    PATH="/opt/bitnami/python/bin:/opt/bitnami/java/bin:/opt/bitnami/spark/bin:/opt/bitnami/spark/sbin:/opt/bitnami/common/bin:$PATH"

COPY prebuildfs /
# Install required system packages and dependencies
RUN install_packages acl ca-certificates curl gzip libbz2-1.0 libc6 libffi6 libgcc1 liblzma5 libncursesw6 libreadline7 libsqlite3-0 libssl1.1 libstdc++6 libtinfo6 procps tar zlib1g
#RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "python" "3.6.13-3" --checksum 24afa03a5424804510c07bb5283eca34df2e065140351714298e5f6c3e995ff7
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "python" "3.8.8-1" --checksum 24afa03a5424804510c07bb5283eca34df2e065140351714298e5f6c3e995ff7
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "java" "1.8.292-0" --checksum 700e2d8391934048faefb45b4c3a2af74bc7b85d4c4e0e9a24164d7256456ca2
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "spark" "3.1.1-1" --checksum 8cc5605af3c72f10b7e20b26a7ee42b4d6f67beb6831a29cd7f2b4037de2e830
RUN . /opt/bitnami/scripts/libcomponent.sh && component_unpack "gosu" "1.12.0-2" --checksum 4d858ac600c38af8de454c27b7f65c0074ec3069880cb16d259a6e40a46bbc50
RUN chmod g+rwX /opt/bitnami

COPY rootfs /
RUN /opt/bitnami/scripts/spark/postunpack.sh
ENV BITNAMI_APP_NAME="spark" \
    BITNAMI_IMAGE_VERSION="3.1.1-debian-10-r45" \
    JAVA_HOME="/opt/bitnami/java" \
    LD_LIBRARY_PATH="/opt/bitnami/python/lib/:/opt/bitnami/spark/venv/lib/python3.6/site-packages/numpy.libs/:$LD_LIBRARY_PATH" \
    LIBNSS_WRAPPER_PATH="/opt/bitnami/common/lib/libnss_wrapper.so" \
    NSS_WRAPPER_GROUP="/opt/bitnami/spark/tmp/nss_group" \
    NSS_WRAPPER_PASSWD="/opt/bitnami/spark/tmp/nss_passwd" \
    SPARK_HOME="/opt/bitnami/spark"

WORKDIR /opt/bitnami/spark
USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/spark/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/spark/run.sh" ]
$ docker build -t 192.168.77.228/yms-system/spark:v3.1.1 .
$ docker push 192.168.77.228/yms-system/spark:v3.1.1

$ docker build -t 192.168.77.228/yms-system/spark-ui-proxy:1.0 .
$ docker push 192.168.77.228/yms-system/spark-ui-proxy:1.0

$ kubectl get pods -n spark
NAME                           READY   STATUS    RESTARTS   AGE
spark-client-578c47b86-rc5wc   1/1     Running   0          5h51m
spark-master-0                 1/1     Running   0          17h
spark-worker-0                 1/1     Running   0          17h
spark-worker-1                 1/1     Running   0          17h

$ kubectl exec -it spark-master-0 -n spark bash
I have no name!@spark-master-0:/opt/bitnami/spark$ python -V
Python 3.8.8

I have no name!@spark-master-0:/opt/bitnami/spark$ ls -al /usr/local/lib/python3.8/site-packages
total 12
drwxr-xr-x 1 root root  217 Apr 10 15:56 .
drwxr-xr-x 1 root root  233 Apr 10 15:55 ..
-rw-r--r-- 1 root root  119 Apr 10 15:55 README.txt
drwxr-xr-x 3 root root   63 Apr 10 15:56 _distutils_hack
-rw-r--r-- 1 root root  152 Apr 10 15:56 distutils-precedence.pth
drwxr-xr-x 5 root root   95 Apr 10 15:56 pip
drwxr-xr-x 2 root root  147 Apr 10 15:56 pip-21.0.1.dist-info
drwxr-xr-x 5 root root   73 Apr 10 15:56 pkg_resources
drwxr-xr-x 7 root root 4096 Apr 10 15:56 setuptools
drwxr-xr-x 2 root root  171 Apr 10 15:56 setuptools-56.0.0.dist-info
drwxr-xr-x 5 root root  209 Apr 10 15:56 wheel
drwxr-xr-x 2 root root  147 Apr 10 15:56 wheel-0.36.2.dist-info
3. Spark chart 설치
Spark master와 worker를 chart로 설치하는 방법을 알아보자.

이 글을 쓰고 있는 시점에 Spark 3.1.1이 설치된다.

// repository add
> helm repo add bitnami https://charts.bitnami.com/bitnami

// spark version 별로 검색
> helm search repo -l spark

// spark chart 5.3.0 download 후 /tmp디렉토리에 압축해제
> helm pull --version 5.3.0 --untar -d /tmp bitnami/spark

// spark namespace에 chart 설치
> helm upgrade -i --create-namespace -n spark my-spark bitnami/spark
자, 이제 spark standalone cluster를 구성하였음으로 sample application을 구동시켜 보자. 아래에서 주목할 것은 deploy-mode=cluster이다.

$ ./bin/spark-submit \
 --class org.apache.spark.examples.SparkPi \
 --master spark://spark-master-svc:7077 \
 --deploy-mode cluster \
 ./examples/jars/spark-examples_2.12-3.1.1.jar 1000
deploy-mode=client 로 할 경우, Spark executor가 driver 주소를 찾지 못해 오류가 발생할 것이다. 이를 해결하기 위해서는 client를 POD에서 기동할 경우 headless service를 생성하고 spark.driver.host 설정을 추가하여 해결할 수 있다.

$ ./bin/spark-submit \
 --class org.apache.spark.examples.SparkPi \
 --master spark://spark-master-svc:7077 \
 --deploy-mode client \
 --conf "spark.driver.host=spark-client-headless" \
 ./examples/jars/spark-examples_2.12-3.1.1.jar 1000
 

 

spark-work 네임스페이스 생성후 k8s에 cluster mode로 deploy하기

kubectl create ns spark-work
kubectl create sa spark -n spark-work
kubectl create clusterrolebinding spark-work-role --clusterrole=edit --serviceaccount=spark-work:spark --namespace=spark-work

spark-submit \
--master "k8s://https://192.168.1.241:6443" \
--deploy-mode cluster \
--name spark-py-pi \
--conf spark.executor.instances=2 \
--conf spark.kubernetes.namespace=spark-work \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.kubernetes.container.image=gcr.io/spark-operator/spark-py:v3.0.0 \
local:///opt/spark/examples/src/main/python/pi.py
4. Spark operator 방식의 chart 설치
CRD를 기반으로 Spark application을 편하게 deploy할 수 있다.

// repository 추가하기
> helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator

// spark-operator namespace에 my-spark-operator 이름으로 deploy하기
> helm upgrade -i -n spark-operator --create-namespace my-spark-operator spark-operator/spark-operator 
아래와 같이 SparkApplication workload에 spark driver와 executor의 instance 개수, resource, image를 설정하여 deploy하면 된다.

// Sample download
> git clone https://github.com/GoogleCloudPlatform/spark-on-k8s-operator.git
> cd spark-on-k8s-operator/
> kubectl apply -f spark-py-pi.yaml

--- spark-py-pi.yaml 내용 --------------------
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: pyspark-pi
  namespace: default
spec:
  type: Python
  pythonVersion: "2"
  mode: cluster
  image: "gcr.io/spark-operator/spark-py:v3.0.0"
  imagePullPolicy: Always
  mainApplicationFile: local:///opt/spark/examples/src/main/python/pi.py
  sparkVersion: "3.0.0"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.0.0
    serviceAccount: spark
  executor:
    cores: 1
    instances: 1
    memory: "512m"
    labels:
      version: 3.0.0
실행후 상태 조회.

$ kubectl get sparkapplication
$ kubectl logs -f yms-spark-spark-operator-79dc778db4-hvhbf -n spark-operator
참고사이트

blog.brainlounge.de/memoryleaks/getting-started-with-spark-on-kubernetes/

 
Bernd's Memory Leaks

« Back to home How to run the first Spark job on a Kubernetes cluster Posted on 2018-12-25 · Tagged in kubernetes, rbac, spark, cloud, , google, gcp, gke · Reading time ~12 minutes · english Apache Spark 2.3 brought initial native support for Kubernete

blog.brainlounge.de
firstheart.tistory.com/m/71

 
Spark on Kubernetes(Google Kubernetes Env) : custom Python

#1. 목표: Python 으로 제작된 Algorithm 소스들을 일정 시간동안 Google Cloud 내에서 실행한다. - 비용등의 문제로 1000건의 Algorithm을 Kubernetes 를 이용하여 동시에 실행처리한다. #2. 이슈: 기존에 개발된..

firstheart.tistory.com
godatadriven.com/blog/spark-kubernetes-argo-helm/



출처: https://cloud2000.tistory.com/entry/Spark-on-kubernetes [클라우드 블로그]