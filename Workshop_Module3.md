# Module 3 : Update Misfits from Kafka Topics

**Time to complete:** 30 minutes

**Services used:**
* [AWS MSK](https://aws.amazon.com/msk/)
* [AWS EC2](https://aws.amazon.com/ec2/)
* [Amazon Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
* [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/)
* [AWS Fargate](https://aws.amazon.com/fargate/)
* [AWS Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)
* [Amazon Cloud9](https://aws.amazon.com/cloud9/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS CodePipeline](https://aws.amazon.com/codepipeline/)
* [AWS CodeDeploy](https://aws.amazon.com/codedeploy/)
* [AWS CodeBuild](https://aws.amazon.com/codebuild/)

## Overview

In Module 3 you will update the Java application to read any updates to different Mysfits from a topic hosted in the [AWS MSK](https://aws.amazon.com/msk/) cluster created during the initial CloudFormation stack build. 

AWS MSK provided the managed service benefits of running a complete Kafka environment. In addition to consuming changes from a topic, you will add messages to the Kafka topic with an [AWS EC2](https://aws.amazon.com/ec2/) instance acting as the Kafka producer.

## Upload KeyPair PEM to Cloud9

At the start of this Workshop you received an email. In that email, grab the link to the PEM file that will be used to SSH into your EC2Kafka Client instance. Run the following command in Cloud9 Terminal to copy the PEM file to your local instance:

```
aws s3 cp s3://INSERT-YOUR-MYMSKKEY-URL ~/environment/myMSKKey.pem
```

## Create Kafka Topics

In your [Amazon Cloud9](https://aws.amazon.com/cloud9/) IDE, ensure that you have a terminal window open. If you do not have one open, you can open a new one under Window -> New Terminal. In the terminal, copy the following commands to [set permissions](https://ss64.com/bash/chmod.html) on the PEM file and SSH into the [AWS EC2](https://aws.amazon.com/ec2/) instance. This instance will act as our Producer to our AWS MSK cluster. The REPLACE_ME_KAFKA_CLIENT_DNS can be found in the cloudformation-core-output.json file.

```
chmod 600 ~/environment/myMSKKey.pem
ssh ec2-user@REPLACE_ME_KAFKA_CLIENT_DNS -i ~/environment/myMSKKey.pem
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
aws kafka describe-cluster --region REPLACE_ME_REGION --cluster-arn=REPLACE_ME_MSK_CLUSTER_ARN > ~/environment/msk-info.json
aws kafka get-bootstrap-brokers --region REPLACE_ME_REGION --cluster-arn=REPLACE_ME_MSK_CLUSTER_ARN > ~/environment/msk-brokers-info.json
```

Open the newly created msk-info.json file. In this file, you will find the metadata for the AWS MSK cluster. Copy the value from the key value "ZookeeperConnectString". The value will look similar to "z-3.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181,z-2.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181,z-1.mskcluster.123456.c5.kafka.us-east-1.amazonaws.com:2181". Paste this value into the following command to create a new topic in Kafka:

```
./kafka/kafka_2.12-2.2.1/bin/kafka-topics.sh --create --zookeeper "REPLACE_ME_ZOOKEEPER_CONNECT_STRING" --replication-factor 2 --partitions 1 --topic misfitupdate
```

With the topic created successfully with the following respone:

```
Created topic misfitupdate.
```

Great! Now you will need some messages on the topic that can be read into the application. To get the Bootstrap server (or brokers) you can open the mskBrokerInfo.json file that we produced earlier.

```
./kafka-console-producer.sh --broker-list REPLACE-ME-BOOTSTRAP-SERVERS --topic misfitupdate
```

With the command run, you will be left with an open session for write messages to the topic (symbol > is shown). Type in the following messages at the prompt:

```
"mysfitId":"4e53920c-505a-4a90-a694-b9300791f0ae","name":"Ben","age":"43"
"mysfitId": "2b473002-36f8-4b87-954e-9a377e0ccbec","name": "Cory","species": "Cargillian"
```

Press Ctrl+C to exit from the Producer script.

We are ready to make changes to the Java application.

## Update the Java Source Code

With the application already integrated with [AWS CodePipeline](https://aws.amazon.com/codepipeline/), we can make modifications to the application and have them continuously deployed to your [AWS Fargate](https://aws.amazon.com/fargate/) cluster. 

In your [Amazon Cloud9](https://aws.amazon.com/cloud9/) IDE, open the file /MythicalMysfitsService-Repository/service/src/main/java/com/example/MythicalMysfitsController.java to update the code file. Copy the entire code segment and overwrite the files contents. Note to replace the REPLACE_ME values with the appropriate values. 

```
package com.example;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import javax.servlet.http.HttpServletResponse;

import org.springframework.stereotype.Service;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import java.util.UUID;
import java.util.Arrays;
import java.util.Properties;

@RestController
public class MythicalMysfitsController {

    @Autowired
    private MythicalMysfitsService mythicalMysfitsService = new MythicalMysfitsService();

    @RequestMapping(value="/mysfits", method=RequestMethod.GET)
    public Mysfits getMysfits(HttpServletResponse response) {
        response.addHeader("Access-Control-Allow-Origin", "*");
        return mythicalMysfitsService.getAllMysfits();
    }

    @RequestMapping(value="/", method=RequestMethod.GET)
    public String healthCheckResponse() {
        return "Nothing here, used for health check. Try /mysfits instead.";
    }
    
    @RequestMapping(value="/kafka", method=RequestMethod.GET)
    public String kafkaCheck(HttpServletResponse response) {
        ConsumerRecords<String, String> recs;
        String retStr = "KAFKA values:<br />";
        
        try {
            Properties props = new Properties();
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "REPLACE_ME_BOOTSTRAP_SERVERS");
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
            props.put(ConsumerConfig.GROUP_ID_CONFIG, "misfit-consumer-group");
            props.put("auto.offset.reset", "earliest");
            props.put("enable.auto.commit", "false");
            
			System.out.printf("Initializing KAFKA CONSUMER");
			KafkaConsumer consumer = new KafkaConsumer(props);
			Thread.sleep(200);
			consumer.subscribe(Arrays.asList("misfitupdate"));
            
            recs = consumer.poll(10000);
            System.out.printf("Done polling Kafka topic...");
            if (recs.count() == 0) {
                retStr += "NO RECORDS<br />";
            } else {
                for (ConsumerRecord<String, String> rec : recs) {
                    retStr += "Recieved " + rec.key() + " " + rec.value() + "<br />";
                }
            }

            consumer.close();
        }
        catch (Exception e) {
            System.out.println("Kafka failure: " + e.getMessage());
        }
        
        response.addHeader("Access-Control-Allow-Origin", "*");
        response.addHeader("Cache-Control", "no-store");
        return retStr;
    }
}
```

You need to inject two other dependencies in this SpringBoot Java application to ensure that it will connect to your AWS MSK cluster. Copy the code snippet below and overwrite the content of the /MythicalMysfitsService-Repository/service/pom.xml. This will ensure that the Kafka dependencies are brought into the application during the Maven build process:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>mythical-mysfits</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.12</artifactId>
            <version>2.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>2.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-streams</artifactId>
            <version>2.2.1</version>
        </dependency>
    </dependencies>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

Once complete with the code changes, we need to commit our changes to Git. This will deploy our changes to our Fargate instance. Run the following commands:

```
git add .
git commit -m "I am going to consume topics from AWS MSK."
git push
```

As stated in Module 2, the change is pushed into the repository, you can open the CodePipeline service in the AWS Console to view your changes as they progress through the CI/CD pipeline. After committing your code change, it will take about 5 to 10 minutes for the changes to be deployed to your live service running in Fargate.  During this time, AWS CodePipeline will orchestrate triggering a pipeline execution when the changes have been checked into your CodeCommit repository, trigger your CodeBuild project to initiate a new build, and retrieve the docker image that was pushed to ECR by CodeBuild and perform an automated ECS [Update Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html) action to connection drain the existing containers that are running in your service and replace them with the newly built image.  

You can view the progress of your code change through the CodePipeline console here (no actions needed, just watch the automation in action!): [AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home)

Open your Mysfits webpage to view the new Kafka RequestMappings contents:

#Replace with your NLB DNS name
http://mysfits-nlb-123456789-abc123456.elb.us-east-1.amazonaws.com/kafka

Next step... integrate the updated messages with the Angular web app!


