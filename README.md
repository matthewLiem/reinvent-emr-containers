# Reinvent - 2019 - Containers on EMR

-----------------------EMR SETUP-----------------------

1) Attach IAM permissions to EMR_EC2_DefaultRole  
Role:EMR_EC2_DefaultRole  
Policy: AmazonEC2ContainerRegistryFullAccess

2) Provision EMR
- emr-6.0.0-beta
- with classification file
- Create/Use PEM file
- VPC with public subnet

```
[{"configurations":[{"classification":"docker","properties":{"docker.privileged-containers.registries":"local,centos,469768379341.dkr.ecr.us-east-1.amazonaws.com","docker.trusted.registries":"local,centos,469768379341.dkr.ecr.us-east-1.amazonaws.com"}}],"classification":"container-executor","properties":{}}]
```

-----------------------DOCKER/ECR SETUP-----------------------

3) SSH to core node
	
4) Docker login
```
aws ecr get-login --region us-east-1 --no-include-email
sudo docker login -u AWS -p <password> https://<account-id>.dkr.ecr.us-east-1.amazonaws.com
```

5) Copy config.json to HDFS
```
# copy it back to user home ~/.docker
mkdir -p ~/.docker
sudo cp /root/.docker/config.json 
sudo chmod 644 ~/.docker/config.json

#push on HDFS such all container can access it
hadoop fs -put ~/.docker/config.json /user/hadoop/
```
6) Create ECR and get endpoint 
  
```
aws ecr create-repository --repository-name emr-docker-examples
```

7) Create docker directories
```
mkdir pyspark
mkdir sparkr
vi pyspark/Dockerfile
vi sparkr/Dockerfile
```

8) Create Image #1 - pyspark - numpy

```
mkdir pyspark
vi pyspark/Dockerfile
```

```
FROM amazoncorretto:8

RUN yum -y update
RUN yum -y install yum-utils
RUN yum -y groupinstall development

RUN yum list python3*
RUN yum -y install python3 python3-dev python3-pip python3-virtualenv

RUN python -V
RUN python3 -V

ENV PYSPARK_DRIVER_PYTHON python3
ENV PYSPARK_PYTHON python3

RUN pip3 install --upgrade pip
RUN pip3 install numpy panda

RUN python3 -c "import numpy as np"
```

9) Create Image #2 - SparkR - randomforest

```
mkdir sparkr
vi sparkr/Dockerfile
```

```
FROM amazoncorretto:8

RUN java -version

RUN yum -y update
RUN amazon-linux-extras enable R3.4

RUN yum -y install R R-devel openssl-devel
RUN yum -y install curl

#setup R configs
RUN echo "r <- getOption('repos'); r['CRAN'] <- 'http://cran.us.r-project.org'; options(repos = r);" > ~/.Rprofile

RUN Rscript -e "install.packages('randomForest')"
  ```

10) Push to ECR

```
sudo docker build -t local/pyspark-example pyspark/
sudo docker tag local/pyspark-example 469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:pyspark-example
sudo docker push 469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:pyspark-example

sudo docker build -t local/sparkr-example sparkr/
sudo docker tag local/sparkr-example 469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:sparkr-example
sudo docker push 469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:sparkr-example
```

-----------------------JOB SUBMISSION-----------------------

11) SSH to the master node

12) Create main.py and sparkR.R

```
vi main.py
```

```
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("docker-numpy").getOrCreate()
sc = spark.sparkContext

import numpy as np
a = np.arange(15).reshape(3, 5)
print(a)
```

```
vi sparkR.R
```

```
library(SparkR)
sparkR.session(appName = "R with Spark example", sparkConfig = list(spark.some.config.option = "some-value"))

sqlContext <- sparkRSQL.init(spark.sparkContext)
library(randomForest)
# check release notes of randomForest
rfNews()

sparkR.session.stop()
```


13) Job submit - pyspark

```
DOCKER_IMAGE_NAME=469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:pyspark-example
DOCKER_CLIENT_CONFIG=hdfs:///user/hadoop/config.json
spark-submit --master yarn \
--deploy-mode cluster \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_TYPE=docker \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE=$DOCKER_IMAGE_NAME \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_CLIENT_CONFIG=$DOCKER_CLIENT_CONFIG \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_MOUNTS=/etc/passwd:/etc/passwd:ro \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_TYPE=docker \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE=$DOCKER_IMAGE_NAME \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_CLIENT_CONFIG=$DOCKER_CLIENT_CONFIG \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_MOUNTS=/etc/passwd:/etc/passwd:ro \
--num-executors 2 \
main.py -v
```

Check job:
```
yarn logs --applicationId application_1574801282656_0023 | grep -C2 '\[\['
```

14) Job submit - SparkR

```
DOCKER_IMAGE_NAME=469768379341.dkr.ecr.us-east-1.amazonaws.com/emr-docker-examples:sparkr-example
DOCKER_CLIENT_CONFIG=hdfs:///user/hadoop/config.json
spark-submit --master yarn \
--deploy-mode cluster \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_TYPE=docker \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE=$DOCKER_IMAGE_NAME \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_CLIENT_CONFIG=$DOCKER_CLIENT_CONFIG \
--conf spark.executorEnv.YARN_CONTAINER_RUNTIME_DOCKER_MOUNTS=/etc/passwd:/etc/passwd:ro \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_TYPE=docker \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_IMAGE=$DOCKER_IMAGE_NAME \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_CLIENT_CONFIG=$DOCKER_CLIENT_CONFIG \
--conf spark.yarn.appMasterEnv.YARN_CONTAINER_RUNTIME_DOCKER_MOUNTS=/etc/passwd:/etc/passwd:ro \
sparkR.R
```
```
yarn logs --applicationId application_1574801282656_0024 | grep -B4 -A10 "Type rfNews"
```
