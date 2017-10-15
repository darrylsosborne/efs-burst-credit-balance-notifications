![](https://s3.amazonaws.com/aws-us-east-1/tutorial/AWS_logo_PMS_300x180.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_available.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_ingergration.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_ecryption-lock.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_fully-managed.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_lowcost-affordable.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_performance.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_scalable.png)![](https://s3.amazonaws.com/aws-us-east-1/tutorial/100x100_benefit_storage.png)
# **Amazon Elastic File System (Amazon EFS)**

## Burst Credit Balance Notifications

### Version 1.0.0

efs-bcbn-1.0.0

---

© 2017 Amazon Web Services, Inc. and its affiliates. All rights reserved. This work may not be  reproduced or redistributed, in whole or in part, without prior written permission from Amazon Web Services, Inc. Commercial copying, lending, or selling is prohibited.

Errors or corrections? Email us at [darrylo@amazon.com](mailto:darrylo@amazon.com).

---

### Overview

This AWS Cloudformation template will create AWS resources to monitor and send notifications if the burst credit balance of the specified Amazon EFS file system has dropped below predefined thresholds.

Throughput on Amazon EFS scales as a file system grows. Because file-based workloads are typically spiky—driving high levels of throughput for short periods of time, and low levels of throughput the rest of the time—Amazon EFS is designed to burst to high throughput levels for periods of time. Amazon EFS uses a credit system to determine when file systems can burst. Each file system earns credits over time at a baseline rate that is determined by the size of the file system, and uses credits whenever it reads or writes data. The baseline rate is 50 MiB/s per TiB of storage (equivalently, 50 KiB/s per GiB of storage). Accumulated burst credits give the file system permission to drive throughput above its baseline rate. When a file system has a positive burst credit balance, it can burst. The burst rate is 100 MiB/s per TiB of storage (equivalently, 100 KiB/s per GiB of storage).

If your workload accessing a file system relies on burst throughput for normal operations, monitoring the file system's burst credit balance is essential. This AWS CloudFormation template will create two Amazon CloudWatch alarms that will send email notifications if the burst credit balance drops below a level where if the file system was being driven at the highest throughput rate possible (permitted throughput), then the burst credit balance would drop to zero in x number of minutes. These minute variables input parameters in the Cloudformation template. One alarm and notification is identified as a 'Warning' and has a default value of 180 minutes. This means that a CloudWatch alarm will send an email notification 180 minutes before the credit balance drops to zero, based on the latest permitted throughput rate. The second alarm and notification is a 'Critical' notification and has a default value of 60 minutes. This alarm will send an email notification 60 minutes before the credit balance drops to zero, based on the latest permitted throughput rate. Permitted throughput is dynamic, scaling up as the file systems grows and scaling down as the file system shrinks. Therefore a third alarm is create that monitors permitted throughput. If the permitted throughput increases by 10%, an email notification is sent and an Auto Scaling Group will launch an EC2 instance that dynamically resets the 'Warning' and 'Critical' alarm burst credit balance thresholds based on the latest permitted throughput rate. This EC2 instance will auto terminate and a new instance will launch only when the permitted throughput rate increases 10%.

This tutorial is designed to help you better understand the performance characteristics of Amazon Elastic File System (Amazon EFS) and how parallelism, I/O size, and Amazon EC2 instance types have a profound effect on file system performance.

This tutorial is divided into four sections. 
>**Section 1** will demonstrate that not all Amazon EC2 instance types are created equal and different instance types provide different levels of network performance when accessing an EFS file system. 
**Section 2** will demonstrate how different I/O sizes (block sizes) and sync() frequencies (the rate data is persisted to disk) have a profound impact on EFS performance when compared to EBS.
**Section 3** will demonstrate how increasing the number of threads accessing EFS will significantly improve performance when compared to EBS.
**Section 4** will compare and demonstrate how different file transfer tools affect performance when accessing an EFS file system.

### Prerequisites

The AWS CloudFormation template below will create the compute environment you need to run the tutorial. You must have an existing Amazon EFS file system in the region where you launch the CloudFormation stack and it must have mount targets in the VPC where you launch your EC2 instances. You will need to provide the EFS file system id as a parameter value when you launch the CloudFormation stack.

### Launch the AWS CloudFormation Stack

Click the  ![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png) link below to create the AWS CloudFormation stack in your account and desired AWS region.

| AWS Region Code | Name | Launch |
| --- | --- | --- 
| us-east-1 |US East (N. Virginia)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=efs-performance-tutorial-'{timestamp}'&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |
| us-east-2 |US East (Ohio)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=efs-performance-tutorial&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |
| us-west-2 |US West (Oregon)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=efs-performance-tutorial&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |
| eu-west-1 |EU (Ireland)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=efs-performance-tutorial&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |
| eu-central-1 |EU (Frankfurt)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/new?stackName=efs-performance-tutorial&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |
| ap-southeast-2 |AP (Sydney)| [![cloudformation-launch-stack](https://s3.amazonaws.com/aws-us-east-1/tutorial/deploy_to_aws_20171004_v2.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/new?stackName=efs-performance-tutorial&templateURL=https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-20170927.yaml) |

There are 

![](https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-ec2-console-screenshot.png)

## Section 1 - Compare the network performance of different EC2 instance types accessing EFS
 ___
This section will demonstrate that not all Amazon EC2 instance types are created equal and different instance types provide different levels of network performance when accessing EFS.


##### 1.1 SSH into all three Amazon EC2 instances

##### 1.2 Use dd to write 20 GB of data to EFS from each instance
Run this command against all three instances to create a 20 GB file on EFS and monitor network traffic and throughput in real-time
```sh
time dd if=/dev/zero of=/efs/tutorial/dd/20G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=1M count=20480 conv=fsync &
nload -u M
```
##### 1.3 Close all SSH sessions
#
#
##### Results
All EC2 instance types have different network performance characteristics so each can drive different levels of throughput to EFS. While the t2.micro instance initially appears to have better network performance when compared against an m4.large instance, it's high network throughput is short lived as a result of the burst characteristics on t2 instances.
| Step | EC2 Instance Type | Data Size | Duration | Burst Throughput | Baseline Throughput | Average Throughput |
| --- | --- | --- | --- | --- | --- | --- |
| 1.2 | t2.micro | 20 GB | 720 seconds | 120 MB/s | 7 MB/s | 30 MB/s |
| 1.2 | m4.large | 20 GB | 384 seconds ||| 56 MB/s |
| 1.2 | c4.2xlarge | 20 GB | 143 seconds ||| 150 MB/s |
## Section 2 - Demonstrate how different I/O sizes and sync frequencies affects throughput to EFS
 ___
This section will compare how different I/O sizes (block sizes) and sync frequencies (the rate data is persisted to disk) have a profound impact on performance between EBS and EFS.

##### 2.1 SSH into the c4.2xlarge EC2 instance
##### 2.2 Write to EBS using 1 MB block size and sync once after each file
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EBS using a 1 MB block size and issuing a sync once at the end to ensure everything is written to disk.
```sh
time dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=1M count=2048 status=progress conv=fsync
```
Record run time.
##### 2.3 Write to EFS using 1 MB block size and sync once after each file
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EFS using a 1 MB block size and issuing a sync once at the end to ensure everything is written to disk.
```sh
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=1M count=2048 status=progress conv=fsync
```
Record run time.
##### 2.4 Write to EBS using 16 MB block size and sync once after each file
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EBS using a 16 MB block size and issuing a sync once at the end to ensure everything is written to disk.
```sh
time dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=16M count=128 status=progress conv=fsync
```
Record run time.
##### 2.5 Write to EFS using 16 MB block size and sync once after each file
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EFS using a 16 MB block size and issuing a sync once at the end to ensure everything is written to disk.
```sh
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=16M count=128 status=progress conv=fsync
```
Record run time.
##### 2.6 Write to EBS using 1 MB block size and sync after each block
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EBS using a 1 MB block size and issuing a sync after each block to ensure each block is written to disk.
```sh
time dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=1M count=2048 status=progress oflag=sync
```
Record run time.
##### 2.7 Write to EFS using 1 MB block size and sync after each block
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EFS using a 1 MB block size and issuing a sync after each block to ensure each block is written to disk.
```sh
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=1M count=2048 status=progress oflag=sync
```
Record run time.
##### 2.8 Write to EBS using 16 MB block size and sync after each block
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EBS using a 16 MB block size and issuing a sync after each block to ensure each block is written to disk.
```sh
time dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=16M count=128 status=progress oflag=sync
```
Record run time.
##### 2.9 Write to EFS using 16 MB block size and sync after each block
Run this command against the c4.2xlarge instance and use dd to create a 2 GB file on EFS using a 16 MB block size and issuing a sync after each block to ensure each block is written to disk.
```sh
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N).img bs=16M count=128 status=progress oflag=sync
```
Record run time.
#
#
##### Results
All EC2 instance types have different network performance characteristics so each can drive different levels of throughput to EFS. While the t2.micro instance appears to have better network performance when initially compared to an m4.large instance, it's high network throughput is short lived as a result of the burst characteristics on t2 instances.
| Step | EC2 Instance Type | Operation | Data Size | Block Size | Sync | Storage | Duration | Throughput |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2.2 | c4.2xlarge | Create | 2 GB | 1 MB | After each file | EBS | 17.0 seconds | 126 MB/s |
| 2.3 | c4.2xlarge | Create | 2 GB | 1 MB | After each file | EFS | 10.7 seconds | 201 MB/s |
| 2.4 | c4.2xlarge | Create | 2 GB | 16 MB | After each file | EBS | 17.0 seconds | 126 MB/s |
| 2.5 | c4.2xlarge | Create | 2 GB | 16 MB | After each file | EFS | 10.6 seconds | 202 MB/s |
| 2.6 | c4.2xlarge | Create | 2 GB | 1 MB | After each block | EBS | 17.3 seconds | 124 MB/s |
| 2.7 | c4.2xlarge | Create | 2 GB | 1 MB | After each block | EFS | 85.5 seconds | 25 MB/s |
| 2.8 | c4.2xlarge | Create | 2 GB | 16 MB | After each block | EBS | 16.3 seconds | 132 MB/s |
| 2.9 | c4.2xlarge | Create | 2 GB| 16 MB | After each block | EFS | 23 seconds | 93 MB/s |
## Section 3 - Demonstrate how multi-threaded access improves throughput and IOPS
 ___
This section will demonstrate how increasing the number of threads accessing EFS will significantly improve performance when compared to EBS.

##### 3.1 SSH into the c4.2xlarge EC2 instance
##### 3.2 Write to EBS using 4 threads and sync after each block
Run this command against the c4.2xlarge instance which will use dd to write 2 GB of data to EBS using a 1 MB block size and issuing a sync after each block to ensure everything is written to disk.
```sh
time seq 0 3 | parallel --will-cite -j 4 'dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{}.img bs=1M count=512 oflag=sync'
```
Record run time.
##### 3.3 Write to EFS using 4 threads and sync after each block
Run this command against the c4.2xlarge instance which will use dd to write 2 GB of data to EFS using a 1 MB block size and issuing a sync after each block to ensure everything is written to disk.
```sh
time seq 0 3 | parallel --will-cite -j 4 'dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{}.img bs=1M count=512 oflag=sync'
```
Record run time.
##### 3.4 Write to EBS using 16 threads and sync after each block
Run this command against the c4.2xlarge instance which will use dd to write 2 GB of data to EBS using a 1 MB block size and issuing a sync after each block to ensure everything is written to disk.
```sh
time seq 0 15 | parallel --will-cite -j 16 'dd if=/dev/zero of=/ebs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{}.img bs=1M count=128 oflag=sync'
```
Record run time.
##### 3.5 Write to EFS using 16 threads and sync after each block
Run this command against the c4.2xlarge instance which will use dd to write 2 GB of data to EFS using a 1 MB block size and issuing a sync after each block to ensure everything is written to disk.
```sh
time seq 0 15 | parallel --will-cite -j 16 'dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{}.img bs=1M count=128 oflag=sync'
```
Record run time.
#
#
##### Results
The distributed data storage design of EFS means that multi-threaded applications can drive substantial levels of aggregate throughput and IOPS. If you parallelize your writes to EFS by increasing the number of threads, you can increase the overall throughput and IOPS to EFS.
| Step | Operation | Data Size | Block Size | Threads | Sync | Storage | Duration | Throughput |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 3.2 | Create | 2 GB | 1 MB | 4 | After each block | EBS | 16.6 seconds | 131 MB/s |
| 3.3 | Create | 2 GB | 1 MB | 4 | After each block | EFS | 21.7 seconds | 99 MB/s |
| 3.4 | Create | 2 GB | 1 MB | 16 | After each block | EBS | 16.4 seconds | 131 MB/s |
| 3.5 | Create | 2 GB | 1 MB | 16 | After each block | EFS | 7.9 seconds | 271 MB/s |
## Section 4 - Compare different file transfer tools
 ___
This section will compare and demonstrate how different file transfer tools affect performance when accessing an EFS file system.

##### 4.1 SSH into the c4.2xlarge EC2 instance
##### 4.2 Review the data to transfer
Run this command against the c4.2xlarge instance to view the total size and count of files to be trasnferred.
```sh
du -csh /ebs/tutorial/data-1m/
find /ebs/tutorial/data-1m/. -type f | wc -l
```
##### 4.3 Set the $instanceid variable
Run this command against the c4.2xlarge instance to set the $instanceid variable which will be used in the preceeding steps.
```sh
instanceid=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```
##### 4.4 Transfer files from EBS to EFS using ***rsync***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using rsync.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time rsync -r /ebs/tutorial/data-1m/ /efs/tutorial/rsync/${instanceid} &
nload -u M
```
##### 4.5 Transfer files from EBS to EFS using ***cp***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using cp.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time cp -r /ebs/tutorial/data-1m/* /efs/tutorial/cp/${instanceid} &
nload -u M
```
##### 4.6 Set the $threads variable
Run this command against the c4.2xlarge instance to set the $threads variable to 4 threads per vcpu. This variable will be used in the subsequent steps.
```sh
threads=$(($(nproc --all) * 4))
```
##### 4.7 Transfer files from EBS to EFS using ***fpsync***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using fpsync.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time /usr/local/bin/fpsync -n ${threads} -v /ebs/tutorial/data-1m/ /efs/tutorial/fpsync/${instanceid} &
nload -u M
```
##### 4.8 Transfer files from EBS to EFS using ***mcp***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using mcp.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time mcp -r --threads=${threads} /ebs/tutorial/data-1m/* /efs/tutorial/mcp/${instanceid} &
nload -u M
```
##### 4.9 Transfer files from EBS to EFS using ***cp + GNU Parallel***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using cp + GNU Parallel.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time find /ebs/tutorial/data-1m/. -type f | parallel --will-cite -j ${threads} cp {} /efs/tutorial/parallelcp &
nload -u M
```
##### 4.10 Transfer files from EBS to EFS using ***fpart + cpio + GNU Parallel***
Run this command against the c4.2xlarge instance to drop caches and transfer 5,000 files (~1 MB each) totalling 5 GB from EBS to EFS using fpart + cpio + GNU Parallel.
```sh
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time /usr/local/bin/fpart -Z -n 1 -o /home/ec2-user/fpart-files-to-transfer /ebs/tutorial/data-1m
head /home/ec2-user/fpart-files-to-transfer.0
time parallel --will-cite -j ${threads} --pipepart --round-robin --block 1M -a /home/ec2-user/fpart-files-to-transfer.0 'sudo cpio -pdm {} /efs/tutorial/parallelcpio/${instanceid}/' &
nload -u M
```
#
#
##### Results
Not all file transfer utilities are created equal. File systems are distributed across an unconstrained number of storage servers and this distributed data storage design means that multithreaded applications like fpsync, mcp, and GNU parallel can drive substantial levels of throughput and IOPS to EFS when compared to single-threaded applications.

| Step | File Transfer Tool | File Count | File Size | Total Size | Threads | Duration | Throughput |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 4.4 | rsync | 5000 | 1 MB | 5 GB | 1 | 435 seconds | 11.7 MB/s |
| 4.5 | cp | 5000 | 1 MB | 5 GB | 1 | 329 seconds | 15.6 MB/s |
| 4.7 | fpsync | 5000 | 1 MB | 5 GB | 32 | 210 seconds | 24.4 MB/s |
| 4.8 | mcp | 5000 | 1 MB | 5 GB | 32 | 87 seconds | 58.9 MB/s |
| 4.9 | cp + GNU Parallel | 5000 | 1 MB | 5 GB | 32 | 73 seconds | 70.1 MB/s |
| 4.10 | fpart + cpio + GNU Parallel | 5000 | 1 MB | 5 GB | 32 | 55 seconds | 93 MB/s |

![](https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-tools-results.png)

## Section 5 - Cleanup
Delete all files on the EFS file system that were created during this tutorial and delete the CloudFormation stack so you don't continue to incur additional charges for these resources.
##### 5.1 Delete all files on the EFS file system created during the tutorial
Run this command on the c4.2xlarge instance to delete the /efs/tutorial data.
```sh
sudo rm /efs/tutorial/ -r 
```
##### 5.2 Delete the AWS CloudFormation stack you launched during the tutorial
![](https://s3.amazonaws.com/aws-us-east-1/tutorial/efs-performance-tutorial-delete-cf-stack-screenshot.png)


## Conclusion
The distributed data storage design of Amazon EFS enables high levels of availability, durability, and scalability. This distributed architecture results in a small latency overhead for each file operation. Due to this per-operation latency, overall throughput generally increases as the average I/O size increases, because the overhead is amortized over a larger amount of data. Amazon EFS supports highly parallelized workloads (for example, using concurrent operations from multiple threads and multiple Amazon EC2 instances), which enables high levels of aggregate throughput and operations per second.

For feedback, suggestions, or corrections, please email me at [darrylo@amazon.com](mailto:darrylo@amazon.com).
