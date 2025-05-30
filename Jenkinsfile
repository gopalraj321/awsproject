pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        AWS_ACCOUNT_ID = "796973491290"
        ECR_REPO = "my-ecr-repo"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CLUSTER_NAME = "my-ecs-cluster"
        SERVICE_NAME = "my-ecs-service"
        TASK_DEF_NAME = "my-task-def"
        VPC_SUBNETS = "subnet-0108422ca3371950d,subnet-077c53d41418790e1"
        SECURITY_GROUP_ID = "sg-0b62e32c8cffbe091"
        LB_NAME = "my-app-lb"
        TG_NAME = "my-target-group"
    }

    stages {
        stage('Login to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                    docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    def exists = sh(script: "aws ecr describe-repositories --repository-names $ECR_REPO --region $AWS_REGION", returnStatus: true) != 0
                    if (exists) {
                        sh "aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION"
                    }
                }
                sh """
                    docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Create Target Group') {
            steps {
                script {
                    def tgArn = sh(
                        script: """
                            aws elbv2 create-target-group \
                                --name $TG_NAME \
                                --protocol HTTP \
                                --port 80 \
                                --vpc-id $(aws ec2 describe-subnets --subnet-ids ${VPC_SUBNETS} --region $AWS_REGION --query "Subnets[0].VpcId" --output text) \
                                --target-type ip \
                                --region $AWS_REGION \
                                --query 'TargetGroups[0].TargetGroupArn' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()
                    env.TARGET_GROUP_ARN = tgArn
                }
            }
        }

        stage('Create Load Balancer') {
            steps {
                script {
                    def lbArn = sh(
                        script: """
                            aws elbv2 create-load-balancer \
                                --name $LB_NAME \
                                --subnets $VPC_SUBNETS \
                                --security-groups $SECURITY_GROUP_ID \
                                --scheme internet-facing \
                                --type application \
                                --region $AWS_REGION \
                                --query 'LoadBalancers[0].LoadBalancerArn' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()
                    env.LOAD_BALANCER_ARN = lbArn
                }
            }
        }

        stage('Create Listener') {
            steps {
                sh """
                    aws elbv2 create-listener \
                        --load-balancer-arn $LOAD_BALANCER_ARN \
                        --protocol HTTP \
                        --port 80 \
                        --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN \
                        --region $AWS_REGION
                """
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                script {
                    def taskDef = """
                    {
                        "family": "$TASK_DEF_NAME",
                        "networkMode": "awsvpc",
                        "containerDefinitions": [
                            {
                                "name": "my-container",
                                "image": "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG",
                                "essential": true,
                                "portMappings": [
                                    {
                                        "containerPort": 80,
                                        "protocol": "tcp"
                                    }
                                ]
                            }
                        ],
                        "requiresCompatibilities": ["FARGATE"],
                        "cpu": "256",
                        "memory": "512"
                    }
                    """
                    writeFile file: 'taskdef.json', text: taskDef
                    sh "aws ecs register-task-definition --cli-input-json file://taskdef.json --region $AWS_REGION"
                }
            }
        }

        stage('Create ECS Cluster') {
            steps {
                script {
                    def exists = sh(script: "aws ecs describe-clusters --clusters $CLUSTER_NAME --region $AWS_REGION", returnStatus: true) != 0
                    if (exists) {
                        sh "aws ecs create-cluster --cluster-name $CLUSTER_NAME --region $AWS_REGION"
                    }
                }
            }
        }

        stage('Create or Update ECS Service') {
            steps {
                script {
                    def exists = sh(script: "aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME --region $AWS_REGION --query 'services[0].status' --output text", returnStatus: true) == 0

                    if (!exists) {
                        sh """
                        aws ecs create-service \
                          --cluster $CLUSTER_NAME \
                          --service-name $SERVICE_NAME \
                          --task-definition $TASK_DEF_NAME \
                          --launch-type FARGATE \
                          --desired-count 1 \
                          --network-configuration 'awsvpcConfiguration={subnets=[$VPC_SUBNETS],securityGroups=[$SECURITY_GROUP_ID],assignPublicIp=ENABLED}' \
                          --load-balancers 'targetGroupArn=$TARGET_GROUP_ARN,containerName=my-container,containerPort=80' \
                          --region $AWS_REGION
                        """
                    } else {
                        sh """
                        aws ecs update-service \
                          --cluster $CLUSTER_NAME \
                          --service $SERVICE_NAME \
                          --task-definition $TASK_DEF_NAME \
                          --region $AWS_REGION
                        """
                    }
                }
            }
        }
    }
}
