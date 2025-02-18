@Library('github.com/releaseworks/jenkinslib') _
pipeline {
    agent any

    stages {
        stage('Clone GitHub Repo') { // Cloning the repository using stored credentials 
            steps {
                git credentialsId: 'github-key', 
                    url: 'https://github.com/mostafamedhat1983/ODC-project.git', 
                    branch: 'main'
            }
        }

        stage('Build Docker Image') { // Building the Docker image
            steps {
                sh 'docker image build -t mostafamedhat1/odc-image:latest .' 
            }
        }

        stage('Push Image to Docker Hub') { // Logging in and pushing the image to docker hub using stored credentials
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', 
                    usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'docker login -u ${USER} -p ${PASS}' 
                    sh 'docker image push mostafamedhat1/odc-image:latest' 
                }
            }
        }

        stage('Create Security Group and Config its Ports') {
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', 
                    credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    
                    // Creating a new security group
                    sh '''aws ec2 create-security-group \
                        --group-name weather-app-SG \
                        --description "Weather app security group" \
                        --vpc-id vpc-0803b8decccf63106 --region us-east-1'''
                    
                    // Opening required ports (SSH, HTTP, HTTPS)
                    sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 22 --cidr 0.0.0.0/0 --region us-east-1'
                    sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1'
                    sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 443 --cidr 0.0.0.0/0 --region us-east-1'
                }
            }
        }

        stage('Create EC2 Instances') { // Launching EC2 instances
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', 
                    credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    // getting security group id as you cant use its name when specifying subnet in ec2 creation
                    // "The parameter groupName cannot be used with the parameter subnet" 
                    sh ''' SG_ID=$(aws ec2 describe-security-groups \
                        --filters Name=group-name,Values=weather-app-SG \
                        --query 'SecurityGroups[0].GroupId' --region=us-east-1 --output text)

                    aws ec2 run-instances --image-id ami-04b4f1a9cf54c11d0 \
                        --count 1 --instance-type t2.micro --key-name Weather-app \
                        --security-group-ids ${SG_ID} --subnet-id subnet-03152cbc82cc6c076 \
                        --region=us-east-1 \
                        --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"server1"}]}' 

                    aws ec2 run-instances --image-id ami-04b4f1a9cf54c11d0 \
                        --count 1 --instance-type t2.micro --key-name Weather-app \
                        --security-group-ids ${SG_ID} --subnet-id subnet-0c79361b53646bed2 \
                        --region=us-east-1 \
                        --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"server2"}]}' 
                    ''' 
                }
            }
        }

        stage('Create Ansible Inventory') { // Generating Ansible inventory file
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', 
                    credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    
                    sh ''' EC2_IP1=$(aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=server1" \
                        --query "Reservations[*].Instances[*].PublicIpAddress" \
                        --region=us-east-1 --output text)

                        EC2_IP2=$(aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=server2" \
                        --query "Reservations[*].Instances[*].PublicIpAddress" \
                        --region=us-east-1 --output text)

                        echo "[webservers]
                        ${EC2_IP1} ansible_user=ubuntu ansible_private_key_file=/var/lib/jenkins/Weather-app.pem
                        ${EC2_IP2} ansible_user=ubuntu ansible_private_key_file=/var/lib/jenkins/Weather-app.pem" > inventory 
                    '''
                }
            }
        }

        stage('Wait Until EC2s Are Up') { // Ensuring EC2 instances are ready
            steps { 
                sh 'sleep 60'  
            }
        }

        stage('Run Ansible Playbook') { // Running the Ansible YAML file
            steps {
                sh 'chmod 600 /var/lib/jenkins/Weather-app.pem'
                sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory docker-install.yaml' 
            }
        }
        stage('Create target group and assign EC2s to it ,create load balancer and listener  ') { 
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', 
                    credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                sh '''
		aws elbv2 create-target-group \
    		--name weather-app-tg \
    		--protocol HTTP \
    		--port 80 \
    		--vpc-id vpc-0803b8decccf63106 \
    		--target-type instance \
    		--health-check-protocol HTTP \
    		--health-check-path / \
    		--health-check-port 80 --region=us-east-1

		EC2_ID1=$(aws ec2 describe-instances \
        --filters "Name=tag:Name,Values=server1" "Name=instance-state-name,Values=running" \
        --query "Reservations[].Instances[].InstanceId" \
        --region us-east-1 \
        --output text )

		EC2_ID2=$(aws ec2 describe-instances \
        --filters "Name=tag:Name,Values=server2" "Name=instance-state-name,Values=running" \
        --query "Reservations[].Instances[].InstanceId" \
        --region us-east-1 \
        --output text )

		TG_ARN=$(aws elbv2 describe-target-groups \
         	--names weather-app-tg --query 'TargetGroups[0].TargetGroupArn' \
         	--region=us-east-1 --output text )

		aws elbv2 register-targets \
        --target-group-arn $TG_ARN \
        --region us-east-1 \
        --targets Id=$EC2_ID1 Id=$EC2_ID2
        
        SG_ID=$(aws ec2 describe-security-groups \
                        --filters Name=group-name,Values=weather-app-SG \
                        --query 'SecurityGroups[0].GroupId' --region=us-east-1 --output text)

        aws elbv2 create-load-balancer \
        --name Weather-app-LB \
        --subnets subnet-03152cbc82cc6c076 subnet-0c79361b53646bed2 \
        --security-groups $SG_ID --region=us-east-1
    
        LB_ARN=$(aws elbv2 describe-load-balancers \
        --names Weather-app-LB --query 'LoadBalancers[0].LoadBalancerArn' \
        --output text --region=us-east-1)
    
        aws elbv2 create-listener \
        --load-balancer-arn $LB_ARN\
        --protocol HTTP \
        --port 80 \
        --default-actions Type=forward,TargetGroupArn=$TG_ARN \
        --region us-east-1
    
                   ''' }               
            }
        }
        stage('Send email with success or failure status') {
            steps { 
                echo "sending email"
            }
    
        post {
            success {
                // sh'''LB_DNS = $(aws elbv2 describe-load-balancers --names Weather-app-LB --query 'LoadBalancers[0].DNSName' --region us-east-1 --output text)'''
                emailext attachLog: true, body: 'Pipeline has successed ', subject: '$BUILD_STATUS! - $BUILD_NUMBER # Build -$PROJECT_NAME', to: 'mostafamomo@gmail.com'
            }    
            failure{
                
                emailext attachLog: true, body: 'Pipeline has failed, pleasecheck log for erorrs ', subject: '$BUILD_STATUS! - $BUILD_NUMBER # Build -$PROJECT_NAME', to: 'mostafamomo@gmail.com'
            }  
        }
    }
}
}
