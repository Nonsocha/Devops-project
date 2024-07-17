```
#Environment variables

ENVIRONMENT=$1

check_num_of_args() {

#Checking the number of arguments

if [ "$#" -ne 0 ]; then

    echo "Usage: $0 <environment>"

    exit 1

fi

}

activate_infra_environment() {

#Acting based on the argument value

if [ "$ENVIRONMENT" == "local" ]; then

  echo "Running script for Local Environment..."

elif [ "$ENVIRONMENT" == "testing" ]; then

  echo "Running script for Testing Environment..."

elif [ "$ENVIRONMENT" == "production" ]; then

  echo "Running script for Production Environment..."

else

  echo "Invalid environment specified. Please use 'local', 'testing', or 
  'production'."
  
  exit 2
fi

}

#Function to check if AWS CLI is installed

check_aws_cli() {

    if ! command -v aws &> /dev/null; then

        echo "AWS CLI is not installed. Please install it before proceeding."

        return 1

    fi

}

#Function to check if AWS profile is set

check_aws_profile() {

    if [ -z "$AWS_PROFILE" ]; then

        echo "AWS profile environment variable is not set."

        return 1

    fi
}

#Array declaration

iam_user(){
    
    # Array of user names

    names=("ken" "john" "kevin" "oyo" "bode")

    # Loop through the array and create IAM users on Aws
    
    for users in "${names[@]}"; do
    
        # Create IAM users
    
        aws iam create-user --user-name "$users"
    
        if [ $? -eq 0 ]; then
    
            echo "IAM users $users created"
    
        else
    
            echo "Failed to create users $users"
    
        fi
    
    done
}

#Function to create an IAM group

create_iam_group() {

     group_name="Admin"

    # Check if group name is provided

    if [ -z "$group_name" ]; then

        echo "Group name is required."

        return 1

    fi

    aws iam create-group --group-name "$group_name"

    # Check if the group was created successfully

    if [ $? -eq 0 ]; then

        echo "IAM group '$group_name' created successfully."

    else

        echo "Failed to create IAM group '$group_name'."

    fi
}

    # Create IAM group using AWS CLI

#Function to attach policy to IAM group

attach_policy_to_group() {

     group_name="Admin"

     policy_arn="arn:aws:iam::aws:policy/AdministratorAccess"


    #Check if group name and policy ARN are provided

    if [ -z "$group_name" ] || [ -z "$policy_arn" ]; then

        echo "Group name and policy ARN are required."

        return 1

    fi

    # Attach policy to IAM group using AWS CLI

    aws iam attach-group-policy --group-name "$group_name" --policy-arn 
    
    "$policy_arn"

    #Check if the policy was attached successfully
    
    if [ $? -eq 0 ]; then
    
        echo "Policy '$policy_arn' attached to group '$group_name' successfully."
    
    else
    
        echo "Failed to attach policy '$policy_arn' to group '$group_name'."
    
    fi
}

#Function to create and deploy Apache on EC2 instances

deploy_apache() {

    # Specify the parameters for the EC2 instances
   
    instance_type="t2.micro"
   
    ami_id="ami-0649bea3443ede307"
   
    count=1  # Number of instances to create
   
    region="us-east-2"  # Region to create cloud resources
   
    subnet_id="subnet-0592ac84b15734705"
   
    key_pair="Ohio"
   
    app_files="/c/Users/ADMIN/Desktop/2130_waso_strategy"
   
    private_key_path="/c/Users/ADMIN/Downloads/Ohio.pem"  # Update this path to your private key file
   
    #Create EC2 instance with user data to install Apache and create the directory
   
    instance_ids=$(aws ec2 run-instances \
        --image-id "$ami_id" \
        --instance-type "$instance_type" \
        --count "$count" \
        --associate-public-ip-address \
        --key-name "$key_pair" \
        --subnet-id "$subnet_id" \
        --region "$region" \
        --user-data '#!/bin/bash
        yum update -y
        yum install -y httpd
        mkdir -p /var/www/html/
        chown ec2-user:ec2-user /var/www/html/
        systemctl start httpd
        systemctl enable httpd' \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ApacheServer}]' \
        --query 'Instances[*].InstanceId' \
        --output text)
   
    #Check if the EC2 instance was launched successfully
   
    if [ $? -eq 0 ]; then
        
        echo "EC2 instances launched successfully with Instance IDs: $instance_ids"
    
    else
        echo "Failed to launch EC2 instances."
    
        return 1
    
    fi
   
    #Wait until the instances are running
    
    echo "Waiting for the instances to be in a 'running' state..."
    
    aws ec2 wait instance-running --instance-ids $instance_ids
    #Add a delay to ensure the public DNS is updated
    sleep 30
    #Loop through each instance ID to get the public DNS and deploy files
    for instance_id in $instance_ids; do
        public_dns=$(aws ec2 describe-instances \
            --instance-ids $instance_id \
            --query 'Reservations[*].Instances[*].PublicDnsName' \
            --output text)
        if [ -z "$public_dns" ]; then
            echo "Failed to retrieve public DNS for instance $instance_id, trying public IP address instead..."
            public_ip=$(aws ec2 describe-instances \
                --instance-ids $instance_id \
                --query 'Reservations[*].Instances[*].PublicIpAddress' \
                --output text)
            if [ -z "$public_ip" ]; then
                echo "Failed to retrieve public IP address for instance $instance_id"
                continue
            else
                public_dns=$public_ip
            fi
        fi
        echo "Deploying files to instance $instance_id with DNS $public_dns"
        #Assuming the instance is Amazon Linux and using SSH to copy files
        scp -i "$private_key_path" -r "$app_files" ec2-user@"$public_dns":/var/www/html/
        # Verify if the files were copied successfully
        if [ $? -eq 0 ]; then
            echo "Files deployed successfully to instance $instance_id ($public_dns)"
        else
            echo "Failed to deploy files to instance $instance_id ($public_dns)"
        fi
    done
}

#Function to create S3 buckets for different departments
create_s3_buckets() {
    # Define a company name as prefix
    company="fashion"
    region="us-east-2"
    #Array of department names
    departments=("marketing" "sales" "hr" "operations" "media")

    #Loop through the array and create S3 buckets for each department
    for department in "${departments[@]}"; do
        bucket_name="${company}-${department}-data-bucket"
        # Create S3 bucket using AWS CLI
        aws s3api create-bucket --bucket "$bucket_name" --region "$region" --create-bucket-configuration LocationConstraint="$region"
        if [ $? -eq 0 ]; then
            echo "S3 bucket '$bucket_name' created successfully."
        else
            echo "Failed to create S3 bucket '$bucket_name'."
        fi
    done
}



#Function for arguments

check_num_of_args

activate_infra_environment

check_aws_cli

check_aws_profile

#Call the function to create users
iam_user

#Create the 'Admin' group
create_iam_group

#Call function to Create the 'Admin' group
create_iam_group

#Call function to attach policy
attach_policy_to_group

#Call Function to deploy Apache on an EC2 instance
deploy_apache

#Call the function to create S3 buckets for different departments
create_s3_buckets
```

