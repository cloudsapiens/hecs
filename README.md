[![hecs logo](https://github.com/cloudsapiens/hecs/blob/main/imgs/rsz_hecs.png)](https://github.com/cloudsapiens/hecs) 

```sh
SAP HANA, express edition on Amazon Elastic Container Service
```

This project is about to guide you through the steps needed to run SAP HANA, Express Edition with Amazon Elastic Container Service (ECS). 
AWS services used for this solution:
  - Amazon Elastic Container Repository (```ECR```)
  - Amazon Elastic Container Service (```ECS```)
  - Amazon Elastic File System (```EFS```)
  - Amazon Elastic Cloud Compute (```EC2```)

Source of the SAP HANA, Express Edition (private repository):  [Docker Hub](https://hub.docker.com/_/sap-hana-express-edition)
# Architecture
[![hecs architecture](https://github.com/cloudsapiens/hecs/blob/main/imgs/architecture.png)](https://github.com/cloudsapiens/hecs) 

# About SAP HANA, express edition
```SAP HANA, express edition``` is a streamlined version of the SAP HANA platform which enables developers to jumpstart application development in the cloud or personal computer to build and deploy modern applications that use up to 32GB memory. SAP HANA, express edition includes the in-memory data engine with advanced analytical data processing engines for business, text, spatial, and graph data - supporting multiple data models on a single copy of the data. 
The software license allows for both non-production and production use cases, enabling you to quickly prototype, demo, and deploy next-generation applications using SAP HANA, express edition without incurring any license fees. Memory capacity increases beyond 32GB are available for purchase at the SAP Store.

# Preparation

First let’s prepare the Docker image. Due to the fact that SAP’s repository is a private one, we need to use our Docker Hub credentials to get the image. I’ve launched a small t2.micro instance in my account and did the Docker image tasks there. You can use your own machine as well.

Note: To use Amazon ECR service, you have to install AWS CLI v2.0. 

### Step 0: Install AWS CLI v2.0 on your machine.

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
Unzip the file: 
```sh
unzip awscliv2.zip
```
Check where the current AWS CLI is installed with which command: 
```sh
which aws
```
It should be in 
```sh
/usr/bin/aws
```

Update it with the following command: 
```sh
sudo ./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update
```
Check the version of AWS CLI: 
```sh
aws --version
```

### Step 1: Configure AWS CLI
Please follow the official AWS documentation and [set-up AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).
### Step 2: Create ECR repository 
Follow this [AWS official guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html).

### Step 3: Pull Docker image
Please ensure that Docker is installed on your machine (e.g. yum install docker in case of Amazon Linux)
Execute command: 
```sh 
docker login
```
Enter your DockerHUB credentials and execute command: 

```sh
docker pull store/saplabs/hanaexpress:2.00.045.00.20200121.1
```
Note: Please always check for latest version on [Docker Hub](https://hub.docker.com/_/sap-hana-express-edition).
Execute command: 
```sh 
docker images
```
You’ll see the following one: 

[![hecs architecture](https://github.com/cloudsapiens/hecs/blob/main/imgs/docker.PNG)](https://github.com/cloudsapiens/hecs)

### Step 4: Push to Amazon Elastic Container Repository (ECR)
Retrieve an authentication token and authenticate your Docker client to your registry. Use the AWS CLI: 
```sh 
aws ecr get-login-password --region <YOURAWSREGION> | docker login --username AWS --password-stdin <YOURAWSACCOUNTID>.dkr.ecr.<YOURAWSREGION>.amazonaws.com
```
Tag the image like the following: 
```sh 
docker tag <YOUR IMAGE ID> <YOURAWSACCOUNTID>.dkr.ecr.<YOURAWSREGION>.amazonaws.com/<YOURREPOSITORYNAME>:latest
````

Run the following command to push this image to your newly created AWS repository: 
```sh 
docker push <YOURAWSACCOUNTID>.dkr.ecr.<YOURAWSREGION>.amazonaws.com/<YOURREPOSITORYNAME>:latest
```
At the end, you will see the Docker image in your repository:
[![hecs ecr](https://github.com/cloudsapiens/hecs/blob/main/imgs/ecr2.PNG)](https://github.com/cloudsapiens/hecs)

### Step 5: Create EFS for your Docker container (volume)
Please follow this [AWS official guide](https://docs.aws.amazon.com/efs/latest/ug/gs-step-two-create-efs-resources.html ) to create an EFS file system.
Ensure that you have a mount target based on this [AWS official guide](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html).

### Step 6: Create Task Definiton
Go to Elastic Container Service and open Task Defintion
  - Choose compatibility EC2
  - Enter name of Task Definition
  - Leave Network Mode on default
  - Choose ecsTaskExecutionRole as Task Execution Role or let the process creates it automatically

Scroll down to Volumes and add Volume as shown on the screen shot:
[![hecs efs](https://github.com/cloudsapiens/hecs/blob/main/imgs/efs.PNG)](https://github.com/cloudsapiens/hecs)

Scroll up to Container Definitions and Add Container:
  - Enter a name for your container
  - Copy the Image URI from ECR and paste it to the text box of Image*
  - Set Memory limits to: Soft limit, 512

Enter port mappings (all TCP): ```39017:39017```, ```39041-39045:39041-39045```, ```1128-1129:1128-1129```, ```59013-59014:59013-59014```

Scroll up to Storage and Logging and define the Volume and CloudWatch settings as shown in the following screenshot:
[![hecs volume](https://github.com/cloudsapiens/hecs/blob/main/imgs/ecs-volumes-logging.PNG)](https://github.com/cloudsapiens/hecs)

Select your predefined Volume and enter Container Path ```/hana/mounts```
Enable logging with CloudWatch, setting the flag ```Auto-configure CloudWatch logs```
Scroll down to Resource Limits and set ```NOFILE``` to ```1048576```

Fianally, scroll to the bottom and click on Add.
### Step 7: Add System Controls and Commands via JSON Editor. 
Note: There are some parameters, which have to be added manually via JSON editor.
Add the following one after the portMappings array: 
```json
     "systemControls": [
        {
          "value": "1073741824",
          "namespace": "kernel.shmmax"
        },
        {
          "value": "40000 60999",
          "namespace": "net.ipv4.ip_local_port_range"
        },
        {
          "value": "524288",
          "namespace": "kernel.shmmni"
        },
        {
          "value": "8388608",
          "namespace": "kernel.shmall"
        }
      ],
      "command": [
        "--passwords-url",
        "file:///hana/mounts/password.json",
        "--agree-to-sap-license"
      ],
```
Parameters:
###### SHMMAX: 
 - This parameter defines the maximum size in bytes of a single shared memory segment that a Linux process can allocate in its virtual address space.
 
###### Net.ipv4.ip_local_port_range: 
- This parameter defines the minimum and maximum port a networking connection can use as its source (local) port. This applies to both TCP and UDP connections.

###### SHMMNI: 
 - This parameter sets the system wide maximum number of shared memory segments.

###### SHMALL: 
 - This parameter sets the total amount of shared memory pages that can be used system wide.

###### --passwords-url:
 - to make your system more secure, you specify your own password before you create your container. This is done by creating a json file as opposed to having a default password. The file can be stored locally or on another system accessible by URL. If the file is to be stored locally, store it in the /data/hana directory you created earlier.

###### --agree-to-sap-license:
 - Indicates you agree to the SAP Developer Center Software Developer License Agreement. You can find the SAP End User License Agreement at https://www.sap.com/docs/download/cmp/2016/06/sap-hana-express-dev-agmt-and-exhibit.pdf. 


### Step 8: Create ECS Cluster based on Spot Fleet (or on-demand instances)
In the ECS console, open ```Clusters```
Choose ```EC2 Linux + Networking``` (Resources to be created: Cluster, VPC, Subnets, Auto Scaling group with Linux AMI)

Set the following:
 - Provisioning Model: Spot, 
 - Spot Instance Allocation Strategy: Lowest Price
 - EC2 Instance Types: R4.XLARGE
 - Maximum price: [check pricing history](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances-history.html) 
 - Select Key Pair or create a new one, you will need that to SSH into the ECS Cluster.
 - Select VPC, CIDR block, Subnet 1 and 2, Security Group or let the process create a new one.
 - Finally click on Create

### Step 9: Create a management instance
[Create a basic Amazon Linux 2 based EC2 (t2.micro) instance](https://docs.aws.amazon.com/efs/latest/ug/gs-step-one-create-ec2-resources.html) in the same VPC/Subnet, where your mount target point has been created for your EFS file system. Adjust the security group’s inbound rule to allow traffic on ```TCP/22 (SSH)``` from your IP address and allow ```TCP/2049 (NFS)``` from your management EC2 instance, so we can access EFS volume later.

### Step 10: Mount EFS volume and prepare it to be used as volume for the Container
Once you have set it up, log on to the instance and create folder ```efs``` in the root folder (/) and cd / .
Afterwards execute the following mount command: 
```sh 
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <YOUREFSID>.efs.<YOURAWSREGION>.amazonaws.com:/ efs
```
Create directory structure on the EFS volume: ```/data/hana``` (with mkdir -p command)
In ```/efs/data/hana``` create a ```password.json``` file and fill with the following JSON object:
```json
{
"master_password" : "YOURVERYSECUREPASSWORD-CHANGE THIS"
}
```
Note: This file serves as the master password for your SAP HANA, express edition users. The password must comply with these rules:
 - At least 8 characters
 - At least 1 uppercase letter
 - At least 1 lowercase letter
 - At least 1 number
 - Can contain special characters, but not ` (backtick), $ (dollar sign), \ (backslash), ' (single quote), or " (double quotation marks).
 - Cannot contain dictionary words
 - Cannot contain simplistic or systemic values, like strings in ascending or descending numerical or alphabetical order

Execute the following commands:
```sh 
chown 12000:79 /data/hana
chmod 600 /efs/data/hana/password.json && chown 12000:79 /efs/data/hana/password.json
```
### Step 11: Run new Task 
 Go to ECS service, Clusters:
 - Select your cluster and click on Tasks tab
 - Click on Run new Task select the following:
 - Launch type: EC2
 - Task Definition: 
 - Family: your task definition
 - Revision: latest
 - Cluster: your cluster
 - Number of tasks: 1
 - Click on Run task

### Step 12: Congratulations

If everything goes well, in the CloudWatch logs you’ll see the the message: ```Startup finished```
Now, you can connect to your SAP HANA database

---
### Step 13: (Optional) Create table with HdbSQL command inside the container
 - Connect to your ECS Cluster via SSH (see IP under tab ECS Instances) with:
```sh 
docker ps -a <CONTAINERID> 
```
 - Execute command: 
```sh 
docker exec -ti <YOURCONTAINERID> bash
```
 - Now, you are inside the container (as user: ```hxeadm```)
 - With the command, you can connect to your DB: 
``` sh 
hdbsql -i 90 -d SYSTEMDB -u SYSTEM -p <YOURVERYSECUREPASSWORD> 
```
 - With the following simple SQL statement you can create a column-stored table with one record: 

```sh
CREATE COLUMN TABLE company_leaders (name NVARCHAR(30), position VARCHAR(30));
INSERT INTO company_leaders VALUES ('Test1', 'Lead Architect');
```
