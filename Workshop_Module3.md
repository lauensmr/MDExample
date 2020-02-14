# Module 3 : Update Misfits from Kafka Topics

![Architecture](/images/module-2/architecture-module-2.png)

**Time to complete:** 45 minutes

**Services used:**
* [AWS MSK](https://aws.amazon.com/msk/)
* [AWS EC2](https://aws.amazon.com/ec2/)
* [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/)
* [Amazon Cloud9](https://aws.amazon.com/cloud9/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS CodePipeline](https://aws.amazon.com/codepipeline/)
* [AWS CodeDeploy](https://aws.amazon.com/codedeploy/)
* [AWS CodeBuild](https://aws.amazon.com/codebuild/)

## Overview

In Module 3 you will update the Java application to read any updates to different Mysfits from a topic hosted in the [AWS MSK](https://aws.amazon.com/msk/) cluster created during the initial CloudFormation stack build. 

AWS MSK provided the managed service benefits of running a complete Kafka environment. In addition to consuming changes from a topic, you will add messages to the Kafka topic with an [AWS EC2](https://aws.amazon.com/ec2/) instance acting as the Kafka producer.

## Create Kafka Topics

In your [Amazon Cloud9](https://aws.amazon.com/cloud9/) IDE, ensure that you have a terminal window open. If you do not have one open, you can open a new one under Window -> New Terminal. In the terminal, copy the following commands to [set permissions](https://ss64.com/bash/chmod.html) on the PEM file and SSH into the [AWS EC2](https://aws.amazon.com/ec2/) instance. This instance will act as our Producer to our AWS MSK cluster.

```
chmod 600 ~/environment/myMSKKey.pem
ssh ec2-user@54.88.211.205 -i ~/environment/myMSKKey.pem
```

The Kafka Client EC2 instance has all of necessary packages to connect to AWS MSK. These packages were installed as part of the [User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) in the CloudFormation Template. For reference, the following script was used in CloudFormation to provision all packages upon creation of the EC2 instance:

```
        #!/bin/bash
        yum update -y 
        yum install python3.7 -y
        yum install java-1.8.0-openjdk-devel -y
        yum erase awscli -y
        cd /home/ec2-user
        echo "export PATH=.local/bin:$PATH" >> .bash_profile
        mkdir kafka
        mkdir mm
        cd kafka
        wget https://archive.apache.org/dist/kafka/2.2.1/kafka_2.12-2.2.1.tgz
        tar -xzf kafka_2.12-2.2.1.tgz
        cd /home/ec2-user
        wget https://bootstrap.pypa.io/get-pip.py
        su -c "python3.7 get-pip.py --user" -s /bin/sh ec2-user
        su -c "/home/ec2-user/.local/bin/pip3 install boto3 --user" -s /bin/sh ec2-user
        su -c "/home/ec2-user/.local/bin/pip3 install awscli --user" -s /bin/sh ec2-user
        chown -R ec2-user ./kafka
        chgrp -R ec2-user ./kafka
        chown -R ec2-user ./mm
        chgrp -R ec2-user ./mm
	yum install aws-cli
```

Now that you have an SSH session started, you want to create a Topic to publish messages to. This will be the same topic that the Mystical Misfits Java application will subscribe to. In order to interact with the AWS MSK cluster, we need to know the [Zookpeer](https://zookeeper.apache.org/) endpoints. To retrieve information about the AWS MSK cluster, we can use the describe-cluster CLI command, which requires the ARN (Amazon Resource Name) of the AWS MSK resource. This value is part of the Output that you saved to the cloudformation-core-output.json file. Find the REPLACE_ME_MSK_CLUSTER_ARN description and copy the OutputValue associated. Replace the REPLACE_ME_MSK_CLUSTER_ARN token in the following command with the OutputValue. NOTE: The aws-cli was installed via the User Data launch commands.

```
aws kafka describe-cluster --region us-east-1 --cluster-arn=REPLACE_ME_MSK_CLUSTER_ARN > msk-info.json
```

Open the newly created msk-info.json file. In this file, you will find the metadata for the AWS MSK cluster. Copy the value from the key value "ZookeeperConnectString". The value will look similar to "z-3.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181,z-2.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181,z-1.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181". Paste this value into the following command to create a new topic in Kafka:

```
./kafka/kafka_2.12-2.2.1/bin/kafka-topics.sh --create --zookeeper "REPLACE_ME_ZOOKEEPER_CONNECT_STRING" --replication-factor 2 --partitions 1 --topic misfitupdate
```

With the topic created successfully with the following respone:

```
Created topic misfitupdate.
```

We are ready to make changes to the Java application.

## Update the Java Source Code

With the application already integrated with [AWS CodePipeline](https://aws.amazon.com/codepipeline/), we can make modifications to the application and have them continuously deployed to your [AWS Fargate](https://aws.amazon.com/fargate/) cluster. 

In your [Amazon Cloud9](https://aws.amazon.com/cloud9/) IDE, open the file ....... update the code file to read the follow.

```
for (int i = 0; ... more java code)
```

Once complete with the code changes, we need to commit our changes to Git. This will deploy our changes to our Fargate instance. Run the following commands:

```
git add .
git commit -m "I am going to consume topics from AWS MSK."
git push
```

As stated in Module 2, the change is pushed into the repository, you can open the CodePipeline service in the AWS Console to view your changes as they progress through the CI/CD pipeline. After committing your code change, it will take about 5 to 10 minutes for the changes to be deployed to your live service running in Fargate.  During this time, AWS CodePipeline will orchestrate triggering a pipeline execution when the changes have been checked into your CodeCommit repository, trigger your CodeBuild project to initiate a new build, and retrieve the docker image that was pushed to ECR by CodeBuild and perform an automated ECS [Update Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html) action to connection drain the existing containers that are running in your service and replace them with the newly built image.  Refresh your Mythical Mysfits website in the browser to see that the changes have taken effect.

You can view the progress of your code change through the CodePipeline console here (no actions needed, just watch the automation in action!):
[AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home)




