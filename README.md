# Jenkins Pipeline for Deploying a Weather App

This Jenkins pipeline automates the process of deploying a weather application. The pipeline performs the following tasks:

1. Clone the GitHub repository.
2. Build and push a Docker image.
3. Set up an AWS security group.
4. Launch EC2 instances.
5. Create an Ansible inventory file.
6. Run an Ansible playbook.
7. Set up an AWS Load Balancer.
8. Send an email notification upon success or failure.

## Prerequisites
Ensure the following credentials and resources are available:
- **GitHub Access**: A repository with the application code.
- **Docker Hub Access**: Credentials for pushing Docker images.
- **AWS Access**: IAM user credentials with necessary permissions.
- **Jenkins Plugins**: Install required plugins (`Pipeline`, `Git`, `Docker`, `AWS CLI` support).

## Pipeline Code Breakdown

### 1. Clone GitHub Repository
This stage pulls the latest version of the application code from GitHub.
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
Builds the Docker image and pushes it to Docker Hub.
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
Defines security rules for EC2 instances.
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
Starts two EC2 instances for the application.
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
Generates an inventory file with instance IPs for Ansible automation.
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

### 6. Deploy with Ansible
Runs an Ansible playbook to configure EC2 instances.
```groovy
stage('Run Ansible Playbook') {
    steps {
        sh 'chmod 600 /var/lib/jenkins/Weather-app.pem'
        sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory docker-install.yaml'
    }
}
```

### 7. Create Load Balancer
Configures an AWS Load Balancer to distribute traffic.
```groovy
stage('Create Load Balancer & Target Group') {
    steps {
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh 'aws elbv2 create-load-balancer --name Weather-app-LB --subnets subnet-03152cbc82cc6c076 subnet-0c79361b53646bed2 --security-groups $SG_ID --region=us-east-1'
        }
    }
}
```

### 8. Email Notifications
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



