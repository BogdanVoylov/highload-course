## 1. VPC
```bash
export VPC_ID=$(aws ec2 create-vpc --cidr-block 10.10.0.0/18 \
  --no-amazon-provided-ipv6-cidr-block \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kma-genesis},{Key=Lesson,Value=public-clouds}]' \
  --query Vpc.VpcId --output text)

echo created vpc $VPC_ID
```
## 2. 3 subnets within your VPC
```bash
export SUBNET_1_ID=$(aws ec2 create-subnet --availability-zone us-east-2a --vpc-id="$VPC_ID" \
  --cidr-block 10.10.1.0/24 --query Subnet.SubnetId --output text)
export SUBNET_2_ID=$(aws ec2 create-subnet --availability-zone us-east-2b --vpc-id="$VPC_ID" \
  --cidr-block 10.10.2.0/24 --query Subnet.SubnetId --output text)
export SUBNET_3_ID=$(aws ec2 create-subnet --availability-zone us-east-2c --vpc-id="$VPC_ID" \
  --cidr-block 10.10.3.0/24 --query Subnet.SubnetId --output text)

echo created subnets $SUBNET_1_ID $SUBNET_2_ID $SUBNET_3_ID
```
## 5. Add to your EC2 instance Security groups, that allows connection to TCP ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
```bash
export SG_ID=$(aws ec2 create-security-group \
  --group-name kma-genesis-sg --description "kma-genesis-sg" \
  --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr  0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr  0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr  0.0.0.0/0
echo created sg $SG_ID
```
## 4. EC2 instance within ASG based on latest AMI Amazon Linux 2 with 15GiB attached EBS (Elastic block storage)
```bash
export LT_NAME=KmaGenesisTemplate
aws ec2 create-launch-template --launch-template-name $LT_NAME \
  --version-description Version1 \
  --launch-template-data '{"NetworkInterfaces":[{"DeviceIndex":0,"AssociatePublicIpAddress":true,"Groups":["'$SG_ID'"],"DeleteOnTermination":true}],"ImageId":"ami-074cce78125f09d61","InstanceType":"t3.micro","BlockDeviceMappings":[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":15}}]}' --region us-east-2
echo created lauch template $LT_NAME
```
## 3. AWS Autoscaling group (ASG)
```bash
export ASG_NAME=KmaGenesisASG
aws autoscaling create-auto-scaling-group --auto-scaling-group-name $ASG_NAME \
  --launch-template "LaunchTemplateName=$LT_NAME" \
  --min-size 1 --max-size 2 --desired-capacity 1 \
  --vpc-zone-identifier "$SUBNET_1_ID,$SUBNET_2_ID,$SUBNET_3_ID" --availability-zones "us-east-2a" "us-east-2b" "us-east-2c"
echo created ASG $ASG_NAME
```
## 6. Put Application Load Balancer (ALB) as a proxy to your ASG
```bash
export ELB_NAME=KmaGenesisLB
aws elbv2 create-load-balancer --name $ELB_NAME  --subnets $SUBNET_1_ID $SUBNET_2_ID $SUBNET_3_ID --security-groups $SG_ID

aws autoscaling attach-load-balancers --load-balancer-names $ELB_NAME --auto-scaling-group-name $ASG_NAME
```
