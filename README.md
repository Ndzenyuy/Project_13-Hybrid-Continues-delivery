# Project 13: Hybrid Continues Delivery of web app

In this project, I implemented a continues delivery of a webapp, this time it is a hybrid deployment(integrating Jenkins with AWS Elastic beanstalk). The CI pipeline builds the artifact and stores in Nexus, this artifact is copied and stored in an S3 bucket from where Beanstalk is configured to pick the same and deploy it to a tomcat platform. Each time a new build is present, the Beanstalk environment automatically updates the running webapp. Two pipelines run here, staging and Production, both hosted as different git branches. Once the staging is good, to lanch production, a simple git merge is required and Jenkins triggers Elastic beanstalk with the coodinates of the artifact running on the staging servers which is then promoted to production.

![](Architecture)

## Prereqs

- Project 5: Continues Integration with Jenkins

## Steps

- S3, IAM and Beanstalk setup
    1. Create User: Goto IAM -> create user

    ```
        name:cicd-bean
        Attach existing polies directly = true
            policies:
                - AdminstratorAccess-AWSElasticBeanstalk
                - AmazonS3FullAccess
        -> Create user
        -> download .csv file
    ```

    Save the credentials in Jenkins: got to Jenkins dashboard -> manage Jenkins -> credentials -> Jenkins -> Global credential:
    ```
        kind: AWS Credentials
        ID: awsbeancreds
        description: awsbeancreds
        Access key ID: <downloaded access key>
        Secret access key: <downloaded secret access key>
        -> create

    ```
    2. Create S3 bucket: goto S3 -> Create bucket
        ```
           Bucket name: vprocicdbean
           region: same region as project
           -> create bucket
        ```
    3. Configure Beanstalk \
        Create an application: ```name: vproapp -> create''' \
        Create a configuration environment:
        ```
            webserver environment = true
            application name: vproapp
            platform: tomcat
            platform branch: Tomcat 8.5 with corretto 11 running on 64bit Amazon Linux 2
            Platform version: 4.3.11
            application code: sample application = true
            presets: custom configuration = true 
                -> next

            Service access: use an existing service role
                existing service role: aws-elasticbeanstalk-service-role
                EC2 key pair: choose a key pair 
                EC2 instance profile: S3full access(if non create one)
                    -> next
            
            VPC: Default VPC
            public IP address: activated = true
            Instance subnets: select all
            security group: select security group which allow http for everywhere
            auto scaling group: Load balanced
                instances: min=2 max = 4
            instance types: t2.micro

            processes: select default -> actions -> edit
                Healthchecks: path = \login




        ```


- Pipeline setup for staging
- Pipeline setup for prod
- CICD flow