# Overview

This document describes how to use spot instances on AWS.  
Spot instances deliver a savings of almost 80% of the on-demand rate.
However they get interrupted, potentially losing data. 

<http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-spot-instances-work.html>

The key to utilizing spot instances is automation, especially around interruption.
A simple workaround, is to use an EBS drive that automounts + a user data script that fires off will allow you to 
take advantage of cheaper spot instances and train your model for weeks at a time.

Your spot bid price determines how often your instance get interrupted.  
Set it low and it will get interrupted moreoften, but you have a firmer handle on price certainity.
Note: the price you pay is the lower of the bid, and the current spot pricing. 
Setting a bid to the on-demand rate would virtually guarantee never getting interrupted.

Finally a S3 bucket is recommended to sync your results.


# Solution

There is some wiring required, that is described here.  I recommend building a stack that you can re-use
for your training jobs.

The stack is shown below.  This document describes how to build parts of this stack.
![CUDA DOCKER AWS](https://www.lucidchart.com/publicSegments/view/b36d7113-4e9a-471c-8cf0-7facf6e17640/image.png)


The current version of this is bare bones.  Further contribution are required.


# Prerequisite:

1) An AMI with Cuda8/Docker/Nvidia-Docker installed
I've made public the following ami in the Oregon Region (us-west-2)

```
ami-f266d292
```

2) Create a Volume with formatted drive (ext4/xfs) in region and snap it.  Record its volume id


3) Docker (optional)...

* If you want to user docker, the AMI is ready to go with Nvidia Docker
https://github.com/NVIDIA/nvidia-docker

* I'd suggest using DockerHub to store containers (its free unless your code is private)
Docker Hub

* A suggested Docker container from Waleed that has tensorflow + opencv is here. Note
start it with nvidia-docker, instead of docker if u want GPU support

<https://hub.docker.com/r/waleedka/modern-deep-learning/>


4) Setup an S3 bucket. (optional)
Nothing special, just to push back models.

For example use s3_parallel to sync your data
```
https://github.com/mishudark/s3-parallel-put
```


# Kick the Tires

0) Launch the AMI and login

1) Run the nvidia-smi to check on running process

```
nvidia-smi

Tue Dec 20 01:25:43 2016
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 367.57                 Driver Version: 367.57                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 0000:00:1E.0     Off |                    0 |
| N/A   61C    P8    30W / 149W |      0MiB / 11439MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

```


2) Start docker and check flags are passed through

```
sudo nvidia-docker run -v /mnt/data:/mnt/data -it nvidia/cuda bash -l
nvcc -V
```


# Run a Spot Instances with persistent request.

0) View spot pricing and region

1) Generate User Data Script (see user_data.sh template)

2) Click on AMI->Spot Request

```
i) GPU types- > pick a GPU Instance (ex. p2.xlarge)
ii) Configure Instace Details-> Spot, 
BidPrice: x.yy
Persistent Request
Network: default
IAMRole:admin
Launch EBS Optimized
Under Advanced Details-> Pick the user_data.sh you've custom modified
iii) Add Storage -> Accept Defaults
iv) Tags -> You user name and task name is useful
vi) Select "Review And Launch".
```

4) Login and view logs

* the AWS startup log
/var/log/cloud-init-output.log

* Docker running 
```
sudo docker ps
sudo docker logs
```

* The data /mnt/data folder where you should be dumping results.
```
df -h
find /mnt/data/
```


# FAQ

1) For tensorflow, or any long running job, how do i not loose my training on interruption?

Use the saver object as described here :
https://www.tensorflow.org/how_tos/variables/

Setup your scripts to routinely dump with a step-id(use utc time), checkpoint every 30 minutes or so
Then on restart the latest checkpoint is picked up.

Note, you will lose some training time, but assuming the AMI stays up for 8 hours, 30 minutes is acceptable as max.

For more durability, upload to S3 incase the EBS fails (rare but can lose all your data)
