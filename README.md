# Jenkins Pipeline for Deploying a Weather App

This Jenkins pipeline automates the process of deploying a weather application. The pipeline performs the following tasks:

1. Clone the GitHub repository.
2. Build and push a Docker image.
3. Set up an AWS security group.
4. Launch 2 EC2 instances in 2 different AZs to ensure availability
5. Create an Ansible inventory file.
6. Run an Ansible playbook.
7. Set up an AWS Load Balancer.
8. Send an email notification upon success or failure.

-Jenkins pipeline is configures to run from a private git hub repo, but this repo visibilty was changed to public for sake of visiblty and sharing

![Untitled Diagram drawio](https://github.com/user-attachments/assets/7533434b-260d-4224-9cb8-974051fd4413)

## Prerequisites
Ensure the following credentials and resources are available:
- **GitHub Access**: A repository with the application code.
- **Docker Hub Access**: Credentials for pushing Docker images.
- **AWS Access**: IAM user credentials with necessary permissions.
- **Jenkins Plugins**: Install required plugins (`Pipeline`, `Git`, `Docker`, `AWS CLI` support).

## Pipeline Code Breakdown

### 1. Clone GitHub Repository
This stage pulls the latest version of the application code from GitHub. and uses the saved credeteinals 'github-key' to authenticate with github 
```groovy
stage('Clone GitHub Repo') {
    steps {
        git credentialsId: 'github-key', 
            url: 'https://github.com/mostafamedhat1983/ODC-project.git', 
            branch: 'main'
    }
}
```

### 2. Build and Push Docker Image
Builds the Docker image and pushes it to Docker Hub. and uses the saved credeteinals 'docker-hub' to authenticate with docker hub 
```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker image build -t mostafamedhat1/odc-image:latest .'
    }
}

stage('Push Image to Docker Hub') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
            sh 'docker login -u ${USER} -p ${PASS}' 
            sh 'docker image push mostafamedhat1/odc-image:latest' 
        }
    }
}
```

### 3. Create AWS Security Group
Creates a security group with name 'weather-app-SG' using saved credentialsId 'aws-key', and put the following ports in the allowed inbound connections  
port 22 to allow ssh connection for remote access  
port 80 to allow http connections  
port 443 to allow https connections ( if needed in the future ) 

```groovy
stage('Create Security Group and Config its Ports') {
    steps {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh '''aws ec2 create-security-group --group-name weather-app-SG \
                --description "Weather app security group" \
                --vpc-id vpc-0803b8decccf63106 --region us-east-1'''
            sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 22 --cidr 0.0.0.0/0 --region us-east-1'
            sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1'
            sh 'aws ec2 authorize-security-group-ingress --group-name weather-app-SG --protocol tcp --port 443 --cidr 0.0.0.0/0 --region us-east-1'
        }
    }
}
```

### 4. Launch EC2 Instances
Create 2 EC2 instances with names server1 and server2 , in two different AZs.  
If you create the EC2 instance using security group name while specifying an avialability zone you will get error   
"The parameter groupName cannot be used with the parameter subnet"  
so you have to use security group ID instead, it is saved in varaible SG_ID   
using saved credentialsId 'aws-key'  

```groovy
stage('Create EC2 Instances') {
    steps {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh '''
            SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=weather-app-SG --query 'SecurityGroups[0].GroupId' --region=us-east-1 --output text)
            
            aws ec2 run-instances --image-id ami-04b4f1a9cf54c11d0 --count 1 --instance-type t2.micro --key-name Weather-app --security-group-ids ${SG_ID} --subnet-id subnet-03152cbc82cc6c076 --region=us-east-1 --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"server1"}]}'
            
            aws ec2 run-instances --image-id ami-04b4f1a9cf54c11d0 --count 1 --instance-type t2.micro --key-name Weather-app --security-group-ids ${SG_ID} --subnet-id subnet-0c79361b53646bed2 --region=us-east-1 --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"server2"}]}'
            '''
        }
    }
}
```

### 5. Create Ansible Inventory
Generates an inventory file with EC2 instance IPs for Ansible automation.  
retrieve the ips of the 2 EC2 instances and then inject them into the ansible inventory file, the private key is already saved on the jenkins host in path /var/lib/jenkins, key owner is changed to jenkins .  
IPs are saved in varaibles EC2_IP1 and EC2_IP2  
using saved credentialsId 'aws-key'  

```groovy
stage('Create Ansible Inventory') {
    steps {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh '''
            EC2_IP1=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=server1" --query "Reservations[*].Instances[*].PublicIpAddress" --region=us-east-1 --output text)
            EC2_IP2=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=server2" --query "Reservations[*].Instances[*].PublicIpAddress" --region=us-east-1 --output text)
            
            echo "[webservers]\n${EC2_IP1} ansible_user=ubuntu ansible_private_key_file=/var/lib/jenkins/Weather-app.pem\n${EC2_IP2} ansible_user=ubuntu ansible_private_key_file=/var/lib/jenkins/Weather-app.pem" > inventory
            '''
        }
    }
}
```
### 7. Ensuring EC2 instances are ready
Wait for 60 seconds to make sure EC2 instances are up and accessible

```groovy
 stage('Wait Until EC2s Are Up') { // Ensuring EC2 instances are ready
            steps { 
                sh 'sleep 60'  
            }
        }
```

### 7. Deploy with Ansible
Runs an Ansible playbook to configure EC2 instances.  
first change key permission to 600 to restrict access to it, then run the ansible playbook 'docker-install.yaml' file 
```groovy
stage('Run Ansible Playbook') {
    steps {
        sh 'chmod 600 /var/lib/jenkins/Weather-app.pem'
        sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory docker-install.yaml'
    }
}
```

This is the 'docker-install.yaml' file
it installs docker, start docker service ,configure Docker to start automatically at boot, pulls the app image from docker hub  
then run the image in the background and forward port 5000 from docker container to port 80 on host to eanble easier url access
```groovy
- name: install docker and run image
  hosts: webservers
  become: true
  tasks:
    - name: Install Docker & Run Image
      shell: |
        apt update
        apt install docker.io -y
        systemctl start docker
        systemctl enable docker
        docker pull mostafamedhat1/odc-image:latest
        docker run -d -p 80:5000 mostafamedhat1/odc-image:latest
```

### 8. Create target group ,assign EC2s to it . Create load balancer and listener

Create a target group named 'weather-app-tg' , the targets in this target group will listen on port 80
```groovy
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
```

Retrieve the 2 EC2 instances IDs and save them in variables EC2_ID1 and EC2_ID2
```groovy
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
```

Retrieve the target group ARN and save it in variable TG_ARN
```groovy
		TG_ARN=$(aws elbv2 describe-target-groups \
         	--names weather-app-tg --query 'TargetGroups[0].TargetGroupArn' \
         	--region=us-east-1 --output text )
```

Register EC2 instances with IDs EC2_ID1 and EC2_ID2 to target group with ARN TG_ARN.   
You have to use target group ARN to register targts ,you cant use its ID or name
```groovy
		aws elbv2 register-targets \
        --target-group-arn $TG_ARN \
        --region us-east-1 \
        --targets Id=$EC2_ID1 Id=$EC2_ID2
```

Save the security group ID in variable SG_ID, and then create load balancer named 'Weather-app-LB' in two different AZs using this security group ID
```groovy
		SG_ID=$(aws ec2 describe-security-groups \
                        --filters Name=group-name,Values=weather-app-SG \
                        --query 'SecurityGroups[0].GroupId' --region=us-east-1 --output text)

        aws elbv2 create-load-balancer \
        --name Weather-app-LB \
        --subnets subnet-03152cbc82cc6c076 subnet-0c79361b53646bed2 \
        --security-groups $SG_ID --region=us-east-1
```

Save load balancer ARN in variable LB_ARN and use it to create a listener which will listen on port 80 and use the target group created before
```groovy
		 LB_ARN=$(aws elbv2 describe-load-balancers \
        --names Weather-app-LB --query 'LoadBalancers[0].LoadBalancerArn' \
        --output text --region=us-east-1)
    
        aws elbv2 create-listener \
        --load-balancer-arn $LB_ARN\
        --protocol HTTP \
        --port 80 \
        --default-actions Type=forward,TargetGroupArn=$TG_ARN \
        --region us-east-1
```


### 9. Email Notifications
Sends an email with the success or failure of the pipeline.
```groovy
post {
    success {
        emailext attachLog: true, body: 'Pipeline has succeeded', subject: '$BUILD_STATUS! - $BUILD_NUMBER # Build -$PROJECT_NAME', to: 'mostafamomo@gmail.com'
    }
    failure {
        emailext attachLog: true, body: 'Pipeline has failed, please check logs for errors', subject: '$BUILD_STATUS! - $BUILD_NUMBER # Build -$PROJECT_NAME', to: 'mostafamomo@gmail.com'
    }
}
```

# Screenshots
### Created EC2 instances
![ec2](https://github.com/user-attachments/assets/0cdb4404-6d46-4012-9821-70a816081309)

### Created Security Group
![security group](https://github.com/user-attachments/assets/7bfd510e-6ac6-4ae1-9f58-2c52f8f744c6)

### Created target Group
![target group](https://github.com/user-attachments/assets/05492a50-6efd-423b-a769-61c1f39d08e1)

### Created load balancer
![load balancer](https://github.com/user-attachments/assets/67672e07-403c-4a8c-8a28-0d1e9da14335)

### Email recieved after pipeline finished
![email](https://github.com/user-attachments/assets/2f0b5dd1-2543-4feb-aada-c64d0b550fd9)

### Weather app accessed from load balancer dns name
![app](https://github.com/user-attachments/assets/7e7b8cb8-659b-4d75-bcbf-ebe6aecccdb5)


# Future enhancements to the pipeline and application
1. replace aws cli commands with terrafrom IAC.
2. create auto scaling group for automatic scaling and management
3. use https instead of http
4. use aws aurora or any RDS database to store Previously Searched Cities as they are now localy stored in each EC2 instance which resultin data discrepancy







