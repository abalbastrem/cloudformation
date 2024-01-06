# README
This project deploys a docker image on Amazon Web Services (aws), assuming the docker has two volumes (conf and log).
It redirects logs onto CloudWatch, which is Amazon's log analysis tool.
It allows to peek inside docker because it's deployed using EC2.

## USER
1. Create AWS user
2. Give permissions to AWS user
2.1. Navigate to IAM.
2.2. In the left menu, click on Users.
2.3. Click on preferred user.
2.4. On the permissions tab, select Add Permissions on the right dropdown.
2.5. Select attach policies directly.
2.6. Select the following policies:
- AmazonEC2FullAccess
- AmazonECS_FullAccess
- AmazonElasticFileSystemFullAccess
- IAMFullAccess

Click next and Add permissions. You should now see them listed in the user permissions tab.

NOTE: The AWS principle for permissions is to assign them as restrictive as possible. The above permissions grant full access to EC2, ECS, EFS and IAM. They do not follow the restrictive principle and are fairly rough, but get the job done for now. For extra security, feel free to go more granular with the permissions.

3. Create access key for said AWS user
3.1. On the user security credentials tab, create an access key.
3.2. Ignore the best practices alternatives, check the confirmation box and just create and access key. Copy the Access Key and the Secret Access Key - you will only have this chance. You will use this to configure the AWS CLI in the following section.

## AWS CLI
1. Install AWS CLI
2. Configure AWS CLI
```bash
aws configure
```
It will prompt four things for you to configure. The first two are the access key you just created for your user.

- AWS Access Key ID
- AWS Secret Access Key
- Default region name i.e. eu-north-1
- Default output format [text|table|json|yaml]

3. Deploy using
```bash
aws cloudformation create-stack --stack-name stack-demo --template-body file://cloudformation.yaml
```

You can know when the deployment has finished deploying
```bash
aws cloudformation wait stack-create-complete --stack-name stack-demo
```
You can check the status of the deployment using
```bash
aws cloudformation describe-stacks --stack-name stack-demo
```
You can delete the stack using
```bash
aws cloudformation delete-stack --stack-name stack-demo
```

NOTE: Updating the stack can be a bit tricky, so it is easier to delete the whole deployment then deploy again. 
NOTE 2: If you delete and deploy again, keep in mind logs and conf files will be reset as EFS volume will be redeployed as well. ECS, IAM and networking entities will be deleted and created anew, but EC2 and EFS instances will die and accumulate. You can navigate to those services and delete them manually.

#### optional: ssh into the running docker
having access to the EC2 allows you to do a series of things, such as modifying the conf files for your task or bug fixing.

1. Make and download EC2 keypair.
Using aws console, go into EC2. On the left margin, click Network & Security > Key Pairs.
Create a key pair with your preferred options. Tried and true options are RSA and PEM. Suggested name is demo_keypair, but you can easily modify it in the cloudformation template, in the field KeyName inside of EC2.

Download the key file and change its permissions to make them more restrictive.
```bash
chmod 400 /path/to/your-key.pem
```
NOTE: ssh might not work if permissions are not this restrictive.

2. Find public IP of deployed EC2 instance:
```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=cf-demo-ec2" --query "Reservations[*].Instances[*].PublicIpAddress" --output text
```

3. Now ssh into your EC2 instance using keypair and public IP.
```bash
ssh -i /path/to/demo_keypair.pem ec2-user@<your-instance-public-ip-or-dns>
```

**Things you can do inside EC2**

A. Modify conf files for task
ssh into running EC2, as described in previous point.
cd into /app/conf.
modify the conf files with vi or nano.
in order for these changes to take effect, you need to reset the task instances (by default, that's just one). The easiest way to do so is using the aws graphical console:
    1. Open the Amazon ECS console at https://console.aws.amazon.com/ecs/.
    2. In the navigation pane, choose "Clusters".
    3. On the "Clusters" page, select the name of the cluster in which your tasks are running.
    4. In the "Services" tab, select the name of the service that is running your tasks.
    5. In the "Tasks" tab, you will see a list of running tasks.
    6. To stop a task, select the checkbox next to the task, then choose "Stop Selected" from the dropdown menu on the right side.
     To stop all of them, simply select "Stop All".
    7. In a few moments, service will automatically spin new task instances that will read from the modified conf files.

WARNING: Do not stop task by stopping docker process in EC2. This will effectively stop the task but service won't spin new instances, so you risk having to deploy everything all over again.

B. If something goes wrong, you can check for errors in these suggested ways.

- To make sure everything gets instanced and paired properly, check logs in
/var/log/cloud-init.log
/var/log/cloud-init-output.log

- Check all processes are running smoothly.
```bash
sudo systemctl status docker
sudo systemctl status ecs
sudo systemctl status amazon-cloudwatch-agent
```

- Checks all packages are installed correctly.
```bash
yum list installed | grep ecs-init
yum list installed | grep amazon-cloudwatch-agent
yum list installed | grep amazon-efs-utils
```

- If that is not enough, you can enter into the running demo docker container. 

Find demo's docker process with
```bash
docker ps
```

Then enter using
```bash
docker exec -ti docker_process_id sh
```

This will open a shell inside the docker process. You can check all files and volumes are in place.
