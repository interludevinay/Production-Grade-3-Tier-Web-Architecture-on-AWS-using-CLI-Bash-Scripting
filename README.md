# ☁️ Production-Grade 3-Tier Web Architecture on AWS using CLI & Bash Scripting
---

# 📘 Project Overview – 3-Tier Web Application Architecture on AWS (Fully Automated via CLI)

This project demonstrates how to deploy a complete, **production-grade 3-Tier Web Architecture on AWS** using only the **AWS Command Line Interface (CLI)** and **Bash scripting**, following **Infrastructure as Code (IaC)** best practices.

The architecture replicates a **real-world scalable web application environment**, with proper separation of concerns, secure network segmentation, and automation-first provisioning — all built end-to-end **without touching the AWS Console**.

---

## ✅ Key Highlights

### ⚙️ Fully Automated Infrastructure Provisioning
Every **VPC, subnet, NAT Gateway, route table, security group, EC2 instance, load balancer**, and **RDS instance** is provisioned using **CLI and Bash** — promoting consistency, reproducibility, and version control.

### 🧩 Scalable and Modular Design
The architecture is built to scale across **multiple Availability Zones**, supporting **high availability** with **Auto Scaling Groups** and **Load Balancers** for the Web and App Tiers.

### 🔐 Network Isolation Between Tiers
A clear network segmentation is established with **dedicated subnets** for the **Web, App, and DB tiers** — enforcing access control and minimizing the attack surface.

---

## 🌐 Real-World Routing Using Route Tables and NAT Gateways

- Each **public subnet** routes to the **Internet Gateway** for inbound/outbound web traffic.
- Each **private App Tier subnet** routes through a **dedicated NAT Gateway** in the same AZ for outbound internet access (package installs, patching, etc.).
- **Private DB subnets** are fully isolated, relying solely on **internal VPC communication** from the App Tier — no internet exposure.

---

## 🔒 Secure Architecture Using VPC, Subnets, SGs, and Load Balancers

- **Custom Security Groups** restrict access between tiers (**Web → App → DB**)
- **Application Load Balancers (ALB)** are used to balance traffic and enhance availability
- **EC2 user data scripts** install and configure services like **NGINX**, enabling zero-manual setup
- **MySQL RDS** is hosted in a **private subnet group** to protect data from external access

---

> 💡 This architecture is suitable for both learning and production-grade environments, offering strong foundations in AWS networking, automation, and layered security.


---

## 🧱 What is a 3-Tier Architecture?

### 1. 🌐 Web Tier (Frontend)
- Hosted in public subnets
- Exposed to the internet via an **Internet-facing Application Load Balancer**
- Serves static/dynamic content
- EC2 instances created via Auto Scaling Group

### 2. 🔧 Application Tier (Backend)
- Deployed in private subnets
- Communicates only with Web Tier and DB Tier
- Auto Scaling Group behind **Internal Load Balancer**
- Uses private routing with outbound internet access (for updates, packages)

### 3. 🗄️ Database Tier
- MySQL (Amazon RDS) hosted in private subnets
- Only accepts traffic from App Tier
- No internet access for security
- Uses DB subnet group with private subnet association

> 🎯 **Benefits**: Scalability, Security, Maintainability

---

## 📷 Architecture Diagram

<img width="900" height="850" alt="Image" src="https://github.com/user-attachments/assets/c5406b98-4e0e-417b-af4d-9ca7eb6546a6" />

---

## 🛠️ Tools & AWS Services Used


| Service                  | Purpose                                     |
|--------------------------|---------------------------------------------|
| AWS CLI + Bash           | Scripting and automation                    |
| Amazon VPC               | Network isolation                           |
| Subnets (Public/Private) | Tier-based segmentation                     |
| Internet Gateway         | Public subnet internet access               |
| Route Tables             | Control routing between tiers               |
| NAT Gateway              | App Tier internet access (lightly integrated) |
| Security Groups          | Controlled traffic flows                    |
| Application Load Balancers| Load balancing Web & App Tier             |
| Launch Templates         | EC2 launch configuration                    |
| Auto Scaling Groups      | Scalable Web & App compute layer            |
| Amazon RDS (MySQL)       | Managed relational database                 |

---

## 🔄 Infrastructure Provisioning Steps

The project follows these structured steps using CLI (complete bash scripting):
### 0. Error Handling
```
#!/bin/bash
set -e #Exit immediately if a command fails

# Trap errors and display a message
trap 'echo "Script failed at line $LINENO. Exiting..."; exit 1' ERR

```

### 1. Create VPC  
```
AZ1="ap-south-1a"
AZ2="ap-south-1b"
```
```
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications ResourceType=vpc,Tags='[{Key=Name,Value=ThreeTierVpc}]'
```
```
VpcId=$(aws ec2 describe-vpcs \
    --filters "Name=tag:Name,Values=ThreeTierVpc" \
    --query "Vpcs[].VpcId" \
    --output text)
```

### 2. Internet Gateway  
- **2a. Create the Internet Gateway**
```
aws ec2 create-internet-gateway \
    --tag-specifications ResourceType=internet-gateway,Tags='[{Key=Name,Value=IGW}]'
```
```
IgwId=$(aws ec2 describe-internet-gateways \
  --query "InternetGateways[1].InternetGatewayId" \
  --output text
)
```
- **2b. Attach the gateway to the VPC**
```
aws ec2 attach-internet-gateway \
    --internet-gateway-id ${IgwId} \
    --vpc-id ${VpcId}
```

### 3. Subnet Creation  
- **3a. WebTierPublicSubnet-1 (Enable auto-assign public IP)**
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ1} \
    --cidr-block 10.0.1.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=PublicWebTierSubnet1}]'
```
```
SubnetId1=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=PublicWebTierSubnet1" \
    --query Subnets[].SubnetId \
    --output text)
```  
```
# Enable auto-assign public IP
aws ec2 modify-subnet-attribute \
  --subnet-id ${SubnetId1} \
  --map-public-ip-on-launch
```

- **3b.WebTierPublicSubnet-2 (Enable auto-assign public IP)**  
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ2} \
    --cidr-block 10.0.2.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=PublicWebTierSubnet2}]'
```
```
SubnetId2=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=PublicWebTierSubnet2" \
    --query Subnets[].SubnetId \
    --output text)
```
```
# Enable auto-assign public IP
aws ec2 modify-subnet-attribute \
  --subnet-id ${SubnetId2} \
  --map-public-ip-on-launch
  ``` 
- **3c. AppTierPrivateSubnet-1**  
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ1} \
    --cidr-block 10.0.3.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=PrivateSubnetTierSubnet1}]'
```
```
PrivateSubnetId1=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=PrivateSubnetTierSubnet1" \
    --query Subnets[].SubnetId \
    --output text)
```
- **3d. AppTierPrivateSubnet-2**  
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ2} \
    --cidr-block 10.0.4.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=PrivateSubnetTierSubnet2}]'
```
```
PrivateSubnetId2=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=PrivateSubnetTierSubnet2" \
    --query Subnets[].SubnetId \
    --output text)
```

```
aws ec2 modify-subnet-attribute \
  --subnet-id ${PrivateSubnetId2} \
  --no-map-public-ip-on-launch
```
- **3e. DbTierPrivateSubnet-1**  
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ1} \
    --cidr-block 10.0.5.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=DbSubnetTierSubnet1}]'
```
```
PrivateDbSubnetId1=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=DbSubnetTierSubnet1" \
    --query Subnets[].SubnetId \
    --output text)
```

- **3f. DbTierPrivateSubnet-2**
```
aws ec2 create-subnet \
    --vpc-id ${VpcId} \
    --availability-zone ${AZ2} \
    --cidr-block 10.0.6.0/24 \
    --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value=DbSubnetTierSubnet2}]'
```
```
PrivateDbSubnetId2=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=DbSubnetTierSubnet2" \
    --query Subnets[].SubnetId \
    --output text)
```

### 4. Create NAT Gateways & Route Tables
- **4a. Create a Public Route Table**
```
aws ec2 create-route-table \
  --vpc-id ${VpcId} \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRT}]'
```
```
PublicRouteTableId=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=PublicRT" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)
```
- **4b. Add Route to Internet Gateway in Public Route Table**

```
aws ec2 create-route \
  --route-table-id ${PublicRouteTableId} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${IgwId}

```
- **4c. Associate Public Route Table with Public Subnets**
```
aws ec2 associate-route-table \
  --subnet-id ${SubnetId1} \
  --route-table-id ${PublicRouteTableId}

aws ec2 associate-route-table \
  --subnet-id ${SubnetId2} \
  --route-table-id ${PublicRouteTableId}
```
- **4d. Create NAT Gateway in AZ-a (Public Subnet 1)**
```
EipAllocationId1=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

NatGatewayId1=$(aws ec2 create-nat-gateway \
  --subnet-id ${SubnetId1} \
  --allocation-id ${EipAllocationId1} \
  --query 'NatGateway.NatGatewayId' \
  --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids ${NatGatewayId1}
```
- **4e. Create NAT Gateway in AZ-b (Public Subnet 2)**
```
EipAllocationId2=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

NatGatewayId2=$(aws ec2 create-nat-gateway \
  --subnet-id ${SubnetId2} \
  --allocation-id ${EipAllocationId2} \
  --query 'NatGateway.NatGatewayId' \
  --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids ${NatGatewayId2}
```
- **Create Private Route Table for AZ-a and Route via NAT Gateway 1**
```
aws ec2 create-route-table \
  --vpc-id ${VpcId} \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRT-A}]'

PrivateRouteTableIdA=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=PrivateRT-A" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)
```
```
aws ec2 create-route \
  --route-table-id ${PrivateRouteTableIdA} \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id ${NatGatewayId1}

aws ec2 associate-route-table \
  --subnet-id ${PrivateSubnetId1} \
  --route-table-id ${PrivateRouteTableIdA}
```
- **4g. Create Private Route Table for AZ-b and Route via NAT Gateway 2**
```
aws ec2 create-route-table \
  --vpc-id ${VpcId} \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRT-B}]'

PrivateRouteTableIdB=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=PrivateRT-B" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)
```
```
aws ec2 create-route \
  --route-table-id ${PrivateRouteTableIdB} \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id ${NatGatewayId2}

aws ec2 associate-route-table \
  --subnet-id ${PrivateSubnetId2} \
  --route-table-id ${PrivateRouteTableIdB}
```

### 5. Create Route Table for DB Tier (No Internet Access)
- **5a. Create DB Private Route Table**
```
aws ec2 create-route-table \
  --vpc-id ${VpcId} \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateDBRT}]'
```
```
PrivateDBRouteTableId=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=PrivateDBRT" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)
```
- **5b. Associate DB Subnets to the DB Route Table**
```
aws ec2 associate-route-table \
  --subnet-id ${PrivateDbSubnetId1} \
  --route-table-id ${PrivateDBRouteTableId}

aws ec2 associate-route-table \
  --subnet-id ${PrivateDbSubnetId2} \
  --route-table-id ${PrivateDBRouteTableId}
```


### 6. Security Groups  
- **6a. WebTierSG – allows HTTP, SSH**
```
aws ec2 create-security-group \
    --group-name WebSG \
    --description "WebTier SG" \
    --vpc-id ${VpcId} \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=WebTierSG}]'
```
```
WebTierSecurityGroupId=$(aws ec2 describe-security-groups \
    --filters "Name=tag:Name,Values=WebTierSG" "Name=vpc-id,Values=${VpcId}" \
    --query SecurityGroups[].GroupId \
    --output text)
```
```
aws ec2 authorize-security-group-ingress \
    --group-id ${WebTierSecurityGroupId} \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```
- **6b. AppTierSG – allows only internal traffic from Web Tier** 
```
aws ec2 create-security-group \
    --group-name AppSG \
    --description "AppTier SG" \
    --vpc-id ${VpcId} \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=AppTierSG}]'
```
```
AppTierSecurityGroupId=$(aws ec2 describe-security-groups \
    --filters "Name=tag:Name,Values=AppTierSG" "Name=vpc-id,Values=${VpcId}" \
    --query SecurityGroups[].GroupId \
    --output text)
```
```
aws ec2 authorize-security-group-ingress \
    --group-id ${AppTierSecurityGroupId} \
    --protocol tcp \
    --port 80 \
    --source-group ${WebTierSecurityGroupId}
```
- **6c. DbTierSG – allows traffic only from App Tier**
```
aws ec2 create-security-group \
    --group-name DbSG \
    --description "DbTier SG" \
    --vpc-id ${VpcId} \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=DbTierSG}]'
```
```
DbTierSecurityGroupId=$(aws ec2 describe-security-groups \
    --filters "Name=tag:Name,Values=DbTierSG" "Name=vpc-id,Values=${VpcId}" \
    --query SecurityGroups[].GroupId \
    --output text)
```
```
aws ec2 authorize-security-group-ingress \
    --group-id ${DbTierSecurityGroupId} \
    --protocol tcp \
    --port 3306 \
    --source-group ${AppTierSecurityGroupId}
```
```
echo "Created Networking part successfully...."
sleep(4)
```
### 7. Target Groups  
```
echo "Creating.... ALB, ASG, RDS part"

```
- **7a. WebTier Target Group**  
```
aws elbv2 create-target-group \
    --name WebTierTG \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id ${VpcId}
```
```
WebTGArn=$(aws elbv2 describe-target-groups \
    --names WebTierTG\
    --query TargetGroups[0].TargetGroupArn \
    --output text)
```
- **7b. AppTier Target Group**
```
aws elbv2 create-target-group \
    --name AppTierTG \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id ${VpcId}
```
```
AppTGArn=$(aws elbv2 describe-target-groups \
    --names AppTierTG\
    --query TargetGroups[0].TargetGroupArn \
    --output text)
```
### 8. Load Balancers  
- **8a. Internet-facing Load Balancer for Web Tier** 
```
aws elbv2 create-load-balancer \
    --name WebTierInternetLB \
    --subnets ${SubnetId1} ${SubnetId2} \
    --security-groups ${WebTierSecurityGroupId} \
    --scheme internet-facing \
    --type application
```
```
WebLBArn=$(aws elbv2 describe-load-balancers \
    --names WebTierInternetLB \
    --query "LoadBalancers[*].LoadBalancerArn" \
    --output text)


```
- **8b. Internal Load Balancer for App Tier**

```
aws elbv2 create-load-balancer \
    --name AppTierInternalLB \
    --scheme internal \
    --subnets ${PrivateSubnetId1} ${PrivateSubnetId2} \
    --security-groups ${AppTierSecurityGroupId} \
    --type application
```
```
AppLBArn=$(aws elbv2 describe-load-balancers \
    --names AppTierInternalLB \
    --query "LoadBalancers[*].LoadBalancerArn" \
    --output text)
```

### 9. Listeners and Routing Rules  
- **9a. Web Tier Listener (e.g., Port 80)**  
```
aws elbv2 create-listener \
    --load-balancer-arn ${WebLBArn} \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=${WebTGArn}
```

- **9b. App Tier Listener (e.g., Port 8080)**
```
aws elbv2 create-listener \
    --load-balancer-arn ${AppLBArn} \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=${AppTGArn}

```

### 10. Launch Templates  
```
WebSGId=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=WebSG" "Name=vpc-id,Values=${VpcId}" \
  --query "SecurityGroups[0].GroupId" \
  --output text)
```
```
AppSGId=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=AppSG" "Name=vpc-id,Values=${VpcId}" \
  --query "SecurityGroups[0].GroupId" \
  --output text)
```
```
USER_DATA=$(base64 -w 0 <<EOF
#!/bin/bash
apt-get update
apt-get install nginx -y
echo "<h1>This is \$(hostname)</h1>" > /var/www/html/index.html
systemctl enable nginx
systemctl start nginx
EOF
)
```
- Define launch configurations for:
  - **Web Tier EC2 Instances**  
  ```
  aws ec2 create-launch-template \
  --launch-template-name WebTierLaunchTemplate \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\": \"ami-0f918f7e67a3323f0\",
    \"InstanceType\": \"t2.micro\",
    \"KeyName\": \"dev\",
    \"SecurityGroupIds\": [\"${WebTierSecurityGroupId}\"],
    \"UserData\": \"${USER_DATA}\"
  }"
  ```
  - **App Tier EC2 Instances**
  ```
  USER_DATA_App=$(base64 -w 0 <<EOF
  #!/bin/bash

  # mysql client
  sudo apt update
  sudo apt install -y mysql-client
  
  # mysql server
  sudo apt install -y mysql-server
  sudo systemctl start mysql
  sudo systemctl enable mysql
  EOF
  )
  ```
  ```
  aws ec2 create-launch-template \
  --launch-template-name AppTierLaunchTemplate \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\": \"ami-0f918f7e67a3323f0\",
    \"InstanceType\": \"t2.micro\",
    \"KeyName\": \"dev\",
    \"SecurityGroupIds\": [\"${AppTierSecurityGroupId}\"],
    \"UserData\": \"${USER_DATA_App}\"
  }"
  ```

### 11. Auto Scaling Groups  
- **11a. ASG for Web Tier**  
```
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name WebASG \
  --launch-template LaunchTemplateName=WebTierLaunchTemplate,Version=1 \
  --min-size 2 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "${SubnetId1},${SubnetId2}" \
  --target-group-arns ${WebTGArn} \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --tags "Key=Name,Value=WebASGInstance,PropagateAtLaunch=true"
```
- **11b. ASG for App Tier**
```
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name AppASG \
  --launch-template LaunchTemplateName=AppTierLaunchTemplate,Version=1 \
  --min-size 1 \
  --max-size 4 \
  --desired-capacity 1 \
  --vpc-zone-identifier "${PrivateSubnetId1},${PrivateSubnetId2}" \
  --target-group-arns ${AppTGArn} \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --tags "Key=Name,Value=AppASGInstance,PropagateAtLaunch=true"
```
- **11c. Scaling policies (based on CPU or network usage)**
```
aws autoscaling put-scaling-policy \
  --policy-name WebTargetTrackingPolicy \
  --auto-scaling-group-name WebASG \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0,
    "DisableScaleIn": false
  }'

```

### 12. RDS Setup  
- **12a. Create a DB Subnet Group (must include 2 subnets in different AZs)**
```
aws rds create-db-subnet-group \
  --db-subnet-group-name rdsDbSG \
  --db-subnet-group-description "Subnet group for DB tier" \
  --subnet-ids ${PrivateDbSubnetId1} ${PrivateDbSubnetId2} \
  --tags Key=Name,Value=DBSubnetGroup

```  
- **12b. Create the RDS instance:**
  - Choose engine (e.g., MySQL, PostgreSQL)
  - Set instance size, DB name, credentials
  - Enable multi-AZ (optional)
  - Configure security group for access from App Tier only
```
aws rds create-db-instance \
  --db-instance-identifier DbTier \
  --allocated-storage 20 \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password admin101 \
  --vpc-security-group-ids ${DbTierSecurityGroupId} \
  --db-subnet-group-name rdsDbSG \
  --no-multi-az \
  --no-publicly-accessible \
  --storage-type gp2 \
  --backup-retention-period 1 \
  --tags Key=Name,Value=Database

```
```
echo "Created Load-Balancer and Auto-Scaling-Group successfully...."
echo "All Commands executed successfully...."
```

---

## 🧾 Project Outcomes

By the end of this setup, you'll have:
- A **fully functional, auto-scalable, and secure cloud-based 3-tier application environment**
- All resources created using **automation scripts (IaC approach)**
- A reusable CLI-based infrastructure deployment workflow


---

## 🧪 AWS 3-Tier Architecture – Testing Guide (Development Only)

---

## 📘 Overview

This section explains how to **test and validate** critical components of the 3-Tier AWS Architecture:

- ✅ **Load Balancer functionality** – Distributes traffic across EC2 instances  
- 📈 **Auto Scaling Group** – Responds to increased load automatically  
- 🛡️ **Database access via NAT Gateway** – For App Tier instances in private subnets

> ⚠️ **Important:**  
> These steps are for **testing and development** purposes only.  
> ⚠️ In a **production** environment, private subnets may **retain NAT Gateway access** (e.g., for package updates or patching),  
> but **Security Group rules must be hardened** to restrict any unnecessary inbound or outbound access, especially from the internet, just for testing purpose.


---

## ✅ 1. Load Balancer Testing (Web Tier)

### 🔍 Goal

Verify that the Internet-facing Load Balancer distributes traffic among Web Tier EC2 instances, which are launched using a Launch Template and part of an Auto Scaling Group.

### 🧪 Steps

1. **Fetch the Load Balancer DNS Name:**

```bash
aws elbv2 describe-load-balancers \
  --names myapp-dev-web-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

2. **Access Load Balancer via Browser or curl:**

```bash
curl http://<Load-Balancer-DNS-Name>
```

3. **if EC2 instances serve different content (e.g., hostname via user data), refreshing the page should cycle through responses.**
```
Example output:

- Served from: ip-10-0-1-10.ec2.internal

- Served from: ip-10-0-2-15.ec2.internal
```

4. **✅ This confirms that:**
```
- Load Balancer is working

- Web Tier instances are healthy and reachable

- Target Group is correctly configured
```

---

## 📈 2. Auto Scaling Group Testing (Web/App Tier)
### 🔍 Goal

Simulate a high-CPU usage scenario and confirm that Auto Scaling Group launches new EC2 instances based on the policy thresholds.

### 🧪 Steps
1. **SSH into a Web Tier EC2 instance:**

```bash
ssh -i myapp-key.pem ec2-user@<Public-IP>
```

2. **Install the stress tool (for Ubuntu Linux):**

```bash
sudo apt install stress
```

3. **Generate high CPU load:**

```bash
stress --cpu 2 --timeout 180
```

4. **Monitor Auto Scaling Activity:**
```
- Navigate to EC2 Console → Auto Scaling Groups

- Check Activity History for scaling actions

- New instances should launch based on the defined scaling policy (e.g., CPU > 70%)

- Wait for cooldown period:

- Once CPU load normalizes, the group will scale in

- Instances will terminate as per scaling-in policy
```


---

## 🛡️ 3.Database Connectivity Testing via NAT Gateway (App Tier → RDS)

### 🔍 Problem

💡 Since the App Tier is deployed in private subnets, it doesn't have internet access by default — which means we can’t directly install packages or run update scripts out of the box.

To solve this, I set up a NAT Gateway to provide secure outbound internet access to the App Tier (for things like installing MySQL/PostgreSQL clients via user data).

For testing, I temporarily adjusted security group rules to allow:

✅ SSH into a Web Tier instance (which is public)

🔄 Then hop from Web Tier into an App Tier instance (private)

This allowed me to verify RDS connectivity, test internal communication, and confirm everything worked exactly as designed — all while keeping the DB tier completely private.

🚨 Note: These changes were made strictly for testing and were rolled back after validation to maintain production-level security.

---

### 🔐 Security Group Configuration

Ensure the following **Security Group (SG)** rules are set up correctly for each tier:

### 🔹 Web Tier SG
- **Inbound Rules**:
  - SSH (port 22) → **Your IP** (`x.x.x.x/32`)
  - HTTP (port 80) → **0.0.0.0/0**

### 🔹 App Tier SG
- **Inbound Rules**:
  - SSH (port 22) → **Web Tier SG**
  - TCP (port 80) → **Web Tier SG**

### 🔹 DB Tier SG
- **Inbound Rules**:
  - MySQL (port 3306) → **App Tier SG**

These rules ensure secure and layered communication between the tiers:
- Only the **Web Tier** is publicly accessible.
- The **App Tier** can be accessed only by the Web Tier.
- The **Database Tier** accepts connections only from the App Tier.
---


### 🧪 Test RDS Connectivity from App Tier

### Step 1: Connect to Bastion (Web Tier) from Your Local Machine

> This instance must have a **public IP** and be accessible over SSH.

```bash
ssh -i <your-key.pem> ec2-user@<Web-Tier-Public-IP>
```

### Step 2: From Web Tier, SSH into App Tier (Private Subnet)

> Use the **private IP** of the App Tier EC2 instance.

📝 **How to find it**:
- Go to AWS Console → EC2 → Select your App instance → Copy its **Private IP**

Then run:

```bash
ssh -i <your-key.pem> ec2-user@<App-Tier-Private-IP>
```
⚠️ **Note**:  
- Use `ec2-user` for Amazon Linux  
- Use `ubuntu` for Ubuntu  
- Use `admin` for other distributions


### Step 3: Connect to RDS from App Tier

#### ✅ For MySQL:

```bash
mysql -h <rds-endpoint> -u <db-username> -p
```

---

## ✅ Confirmation Checklist

- [x] App EC2 can reach the RDS instance.
- [x] RDS Security Group allows **inbound traffic** from the **App Tier Security Group**.
- [x] NAT Gateway is correctly enabling **outbound internet access** from the private subnet (App Tier).

---



