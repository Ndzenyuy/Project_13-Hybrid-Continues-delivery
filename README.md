# Project 13: Hybrid Continues Delivery of web app

In this project, I implemented a continues delivery of a webapp, this time it is a hybrid deployment(integrating Jenkins with AWS Elastic beanstalk). The CI pipeline builds the artifact and stores in Nexus, this artifact is copied and stored in an S3 bucket from where Beanstalk is configured to pick the same and deploy it to a tomcat platform. Each time a new build is present, the Beanstalk environment automatically updates the running webapp. Two pipelines run here, staging and Production, both hosted as different git branches. Once the staging is good, to lanch production, a simple git merge is required and Jenkins triggers Elastic beanstalk with the coodinates of the artifact running on the staging servers which is then promoted to production.

![](https://github.com/Ndzenyuy/Project_13-Hybrid-Continues-delivery/blob/cicd-jenbean-stage/images/Project13_architecture.jpg)

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
                Healthchecks: path = \login  -> save
                -> next

            Rolling updates and deployments
                Deployment Policy: Rolling
                Batch size: percentage -> 50%
                -> next

            Review and Submit,
                This will launch 2 EC2 instances running the sample vode.


        ```

- Pipeline setup for staging \
    Open the project folder on local machine, in the branch ```ci-jenkins``` then create a new branch from it 
    ```git checkout -b cicd-jenbean-stage```

    Open a code editor from terminal by running ```code .``` modify the Jenkinsfile to

    ```
        pipeline {
        agent any
        tools {
            maven "MAVEN3"
            jdk "OracleJDK8"
        }
        
        environment {
            SNAP_REPO = 'vprofile-snapshot'
            NEXUS_USER = 'admin'
            NEXUS_PASS = 'admin123'
            RELEASE_REPO = 'vprofile-release'
            CENTRAL_REPO = 'vpro-maven-central'
            NEXUSIP = '172.31.30.107'
            NEXUSPORT = '8081'
            NEXUS_GRP_REPO = 'vpro-maven-group'
            NEXUS_LOGIN = 'nexuslogin'
            SONARSERVER = 'sonarserver'
            SONARSCANNER = 'sonarscanner'
            ARTIFACT_NAME = "vprofile-v${BUILD_ID}.war"
            AWS_S3_BUCKET = 'vprofile12cicdbean'
            AWS_EB_APP_NAME = 'vproapp'
            AWS_EB_ENVIRONMENT = 'Vproapp-env'
            AWS_EB_APP_VERSION = "${BUILD_ID}"
        }

        stages {
            stage('Build'){
                steps {
                    sh 'mvn -s settings.xml -DskipTests install'
                }
                post {
                    success {
                        echo "Now Archiving."
                        archiveArtifacts artifacts: '**/*.war'
                    }
                }
            }

            stage('Test'){
                steps {
                    sh 'mvn -s settings.xml test'
                }

            }

            stage('Checkstyle Analysis') {
                steps {
                    sh 'mvn -s settings.xml checkstyle:checkstyle'

                }
            }

            stage('CODE ANALYSIS with SONARQUBE') {
            
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }

            steps {
                
                withSonarQubeEnv("${SONARSERVER}") {
                sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"
                }

                

                /*timeout(time: 10, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
                }*/
            }
            } 

            stage("UploadArtifact") {
                steps{
                    nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                    [artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war']
                    ]
                    )
                }
            } 

            stage("Deploy to Stage Bean") {
                steps {
                    withAWS(credentials: 'awsbeancreds', region: 'us-east-2') {
                        sh 'aws s3 cp ./target/vprofile-v2.war s3://$AWS_S3_BUCKET/$ARTIFACT_NAME'
                        sh 'aws elasticbeanstalk create-application-version --application-name $AWS_EB_APP_NAME --version-label $AWS_EB_APP_VERSION --source-bundle S3Bucket=$AWS_S3_BUCKET,S3Key=$ARTIFACT_NAME'
                        sh 'aws elasticbeanstalk update-environment --application-name $AWS_EB_APP_NAME --environment-name $AWS_EB_ENVIRONMENT --version-label $AWS_EB_APP_VERSION'            
                }
                }
            }
            
            /* stage('Slack'){
                steps{
                    slackSend message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                }
            }   */  
        }
    }

    ```

    Create a pipeline in Jenkins: -> New Item
    ```
        name: cicd-jenkins-bean-stage
        pipeline = true
        copy from: vprofile-ci-pipeline
        -> save

        Build triggers: Github hook trigger for GITScm polling = true
        Pipeline:
            Definition: pipeline script from SCM
            SCM: Git
                Repository URL = <copy ssh url of github repo>
                credentials: git(githublogin)
                branch: cicd-jenbean-stage
        -> save


    ```
    On Jenkins dashboard, build the project
    ![](https://github.com/Ndzenyuy/Project_13-Hybrid-Continues-delivery/blob/cicd-jenbean-stage/images/stage-build-success.png)
    Wait for a few minutes and Check the url on elastic beanstalk to the App, the app should be deployed
    ![]()


- Pipeline setup for prod

    Go to elastic beanstalk, select the application and create a new environment
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
                Healthchecks: path = \login  -> save
                -> next

            Rolling updates and deployments
                Deployment Policy: Rolling
                Batch size: percentage -> 50%
                -> next

            Review and Submit,
                This will launch 2 EC2 instances running the sample vode.


        ```
    Create a new branch from cicd-jenbean-stage
        ```
            git checkout -b cicd-jenbean-prod
        ```

        Update the Jenkinsfile for prod to have
        ```
            def buildNumber = Jenkins.instance.getItem('cicd-jenkins-bean-stage').lastSuccessfulBuild.number

            pipeline {
                agent any
                tools {
                    maven "MAVEN3"
                    jdk "OracleJDK8"
                }
                
                environment {
                    SNAP_REPO = 'vprofile-snapshot'
                    NEXUS_USER = 'admin'
                    NEXUS_PASS = 'admin123'
                    RELEASE_REPO = 'vprofile-release'
                    CENTRAL_REPO = 'vpro-maven-central'
                    NEXUSIP = '172.31.30.107'
                    NEXUSPORT = '8081'
                    NEXUS_GRP_REPO = 'vpro-maven-group'
                    NEXUS_LOGIN = 'nexuslogin'
                    SONARSERVER = 'sonarserver'
                    SONARSCANNER = 'sonarscanner'
                    ARTIFACT_NAME = "vprofile-v${buildNumber}.war"
                    AWS_S3_BUCKET = 'vprofile12cicdbean'
                    AWS_EB_APP_NAME = 'vproapp'
                    AWS_EB_ENVIRONMENT = 'Vproapp-prod'
                    AWS_EB_APP_VERSION = "${buildNumber}"
                }

                stages {
                    stage('Deploy to Stage Bean'){
                    steps {
                        withAWS(credentials: 'awsbeancreds', region: 'us-east-2') {
                        sh 'aws elasticbeanstalk update-environment --application-name $AWS_EB_APP_NAME --environment-name $AWS_EB_ENVIRONMENT --version-label $AWS_EB_APP_VERSION'
                        }
                    }
                    }
                    
                    /* stage('Slack'){
                        steps{
                            slackSend message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                        }
                    }   */  
                }
            }

        ```

    Goto Jenkins dashboard, create a new item
    ```
        name: cicd-jenbean-prod
        pipeline = true
        copy from: cicd-jenbean-stage
        -> save

        modify Branches to build from */cicd-jenbean-stage to */cicd-jenbean-prod
        -> save
    ```

    Build pipeline, now our script requires approval from the stage pipeline since we are trying to read it's BUILD_ID, we have to manually approve the prompts for several times by clinking the link as on the log below and approving. The builds will succed after the fourth approval.
    ![](error message awaiting approval)
    ![](approved)

- CICD flow
    Modify the code, by adding comments, then push it to github, from the staging branch. The pipeline should be triggered which will build the pipeline. If everything goes well, it can be promoted to production by running a ```git merge cicd-jenbean-stage``` from the production branch. This will lauch the prod pipeline which will simply deploy the exact same version as in staging, by picking the buildid from the stage pipeline.
