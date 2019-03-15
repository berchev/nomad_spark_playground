# nomad_spark_playground

## Description

Downloading this repo content and following the provided steps, you are going to have installed and configurred nomad-spark environment in AWS. The environment will have:
- 3 Nomad Servers
- 4 Nomad Clients

## Files
| File                   | Description                                                                                                |
|          :---         |                                    :---                                                                |
|`aws/env/us-east`       | Contain terraform code files required to create AWS Nomad Clients and Servers                              | 
|`aws/modules/hashistack`| Contain terraform module file                                                                              |
|`aws/packer.json`       | Packer file needed for creation of AWS image (Later we will use this image to create AWS environment)      |
|`exmples/spark`         | Spark configurations needed for creation of Spark History Server and HDFS (Hadoop distributed file system) |
|`Vagrantfile`           | Vagrant configuration file of our DEV environment                                                          |
|`shared`                | Contain configuration and installation scripts                                                             |
|`nomad-spark-pic`       | Contain example pictures                                                                                   |

## Requirements:
- Virtualbox installed. If not click [here](https://www.virtualbox.org/wiki/Downloads) to download and install
- Vagrant installed. If not click [here](https://www.vagrantup.com/docs/installation/)
- AWS account
  - If you do not have AWS account click [here](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) in order to create one.
  - Set [MFA](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#multi-factor-authentication)
  - Set [Access Key ID and Secret Access Key ](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys)
  - Set [Key Pair](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#key-pairs)

## Instructions on how to use this repo 
- download content of this repo: `git clone https://github.com/berchev/nomad_spark_playground.git`
- change to nomad_spark_playground: `cd nomad_spark_playground`
- do `vagrant up` in order to create dev vagrant environment. (Note that your future work is going to happen from that machine)
- do `vagrant ssh` in order to perform ssh to target vagrant machine. The future steps need to be executed from this machine!
- export aws access and secret key as environbent variables:
```
export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
- change to ~/aws: `cd to ~/aws`
- packer build packer.json in order to buld AWS image. Based on that image you will create your future nomad-spark environment in AWS 
- Once packer build script finish, found the created new image in us-east-1 region. (in my case ami-0384963dd9169550d)
- cd to ~/aws/env/us-east and create terraform.tfvars file as follows:
  - fulfill your ami resulted from packer build command
  - update your KEY_PAIR_NAME (without **.pem** extension)
  ```
  region                  = "us-east-1"
  ami                     = "ami-xxxxxxxxxxxxxxxx"
  instance_type           = "t2.medium"
  key_name                = "KEY_PAIR_NAME"
  server_count            = "3"
  client_count            = "4"
  ```
- terraform init
- terraform plan
- terraform apply (terraform is going to crete 11 resources)
- when terraform provisioning finish you will see green output with all information you need. For example:
```
Outputs:

IP_Addresses = 
Client public IPs: 34.230.25.206, 3.90.177.196, 54.89.152.79, 52.91.222.249
Server public IPs: 34.238.159.72, 35.153.142.29, 54.173.212.111

To connect, add your private key and SSH into any client or server with
`ssh ubuntu@PUBLIC_IP`. You can test the integrity of the cluster by running:

  $ consul members
  $ nomad server members
  $ nomad node status

If you see an error message like the following when running any of the above
commands, it usually indicates that the configuration script has not finished
executing:

"Error querying servers: Get http://127.0.0.1:4646/v1/agent/members: dial tcp
127.0.0.1:4646: getsockopt: connection refused"

Simply wait a few seconds and rerun the command if this occurs.

The Nomad UI can be accessed at http://PUBLIC_IP:4646/ui.
The Consul UI can be accessed at http://PUBLIC_IP:8500/ui.
```
- your nomad-spark environment is ready for use


## Instructions on how to use your nomad-spark environment
- To give the Spark integration a test drive, do SSH to one of the Nomad Servers or one of the Nomad Clients provisioned in AWS with terraform: `ssh -i /path/to/key_pair ubuntu@PUBLIC_IP`
- We are going to deploy HDFS (Hadoop distributed file system). Spark is designed to read from and write to HDFS. We can do that from one of AWS machines, which we just provision with terraform:
  - `cd $HOME/examples/spark`
  - `nomad run hdfs.nomad` in order to run sample HDFS job file
  - `nomad status hdfs` in order to check if all allocations are in running state
- We are going to crete now directories and files in HDFS for use by the history server and the sample Spark jobs
  - `hdfs dfs -mkdir /foo` in order to create dir `foo`
  - `hdfs dfs -put /var/log/apt/history.log /foo` in order to place `history.log` file in `/foo` directory
  - `hdfs dfs -mkdir /spark-events` in order to create `/spark-events` directory
  - `hdfs dfs -ls /` in order to check your work
- Now we can deploy spark history server:
  - `nomad run spark-history-server-hdfs.nomad`
  - You can get the private IP for the history server with a Consul DNS lookup: `nslookup spark-history.service.consul`
  - Once you have the IP, you can access the history server by `http://PUBLIC_IP:18080`
  ![](https://github.com/berchev/nomad_spark_playground/blob/master/nomad-spark-pics/spark-history-server.png)
  
- Let's create some Spark jobs. The `spark-submit commands listed below demonstrate several of the official Spark examples:
  - SparkPi (Java)
  ```
  spark-submit \
  --class org.apache.spark.examples.JavaSparkPi \
  --master nomad \
  --deploy-mode cluster \
  --conf spark.executor.instances=4 \
  --conf spark.nomad.cluster.monitorUntil=complete \
  --conf spark.eventLog.enabled=true \
  --conf spark.eventLog.dir=hdfs://hdfs.service.consul/spark-events \
  --conf spark.nomad.sparkDistribution=https://s3.amazonaws.com/nomad-spark/spark-2.1.0-bin-nomad.tgz \
  https://s3.amazonaws.com/nomad-spark/spark-examples_2.11-2.1.0-SNAPSHOT.jar 100
  ```
  - Word count (Java)
  ```
  spark-submit \
  --class org.apache.spark.examples.JavaWordCount \
  --master nomad \
  --deploy-mode cluster \
  --conf spark.executor.instances=4 \
  --conf spark.nomad.cluster.monitorUntil=complete \
  --conf spark.eventLog.enabled=true \
  --conf spark.eventLog.dir=hdfs://hdfs.service.consul/spark-events \
  --conf spark.nomad.sparkDistribution=https://s3.amazonaws.com/nomad-spark/spark-2.1.0-bin-nomad.tgz \
  https://s3.amazonaws.com/nomad-spark/spark-examples_2.11-2.1.0-SNAPSHOT.jar \
  hdfs://hdfs.service.consul/foo/history.log
  ```
  - DFSReadWriteTest (Scala)
  ```
  spark-submit \
  --class org.apache.spark.examples.DFSReadWriteTest \
  --master nomad \
  --deploy-mode cluster \
  --conf spark.executor.instances=4 \
  --conf spark.nomad.cluster.monitorUntil=complete \
  --conf spark.eventLog.enabled=true \
  --conf spark.eventLog.dir=hdfs://hdfs.service.consul/spark-events \
  --conf spark.nomad.sparkDistribution=https://s3.amazonaws.com/nomad-spark/spark-2.1.0-bin-nomad.tgz \
  https://s3.amazonaws.com/nomad-spark/spark-examples_2.11-2.1.0-SNAPSHOT.jar \
  /etc/sudoers hdfs://hdfs.service.consul/foo
  ```
- you can monitor via terminal your spark jobs via followong commands:
  - `nomad status`
  - `nomad status JOB_ID`
  - `nomad alloc-status DRIVER_ALLOC_ID`
  - `nomad logs DRIVER_ALLOC_ID`
  
  ![](https://github.com/berchev/nomad_spark_playground/blob/master/nomad-spark-pics/nomad-cli1.png)
  ![](https://github.com/berchev/nomad_spark_playground/blob/master/nomad-spark-pics/nomad-cli2.png)
  
- you can use UI too:
  - `Nomad_Server_IP:4646` into browser
  - Check different menus: `Jobs`, `Clients`, `Servers`
  ![](https://github.com/berchev/nomad_spark_playground/blob/master/nomad-spark-pics/nomad1.png)



  ## TODO
