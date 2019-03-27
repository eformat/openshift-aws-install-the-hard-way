## 3.11 OpenShift install on AWS the hard way

Updated for 3.11 install. A Minimal install (all of the bits can be changed):
- choose how many nodes - default 9 node HA cluster (x3 masters, x3 infra, x3 app), min 1,1,1
- choose machine sizes - use `t2.small|t2.medium` for bastion|nodes for minimum install
- choose machine ec2 ebs disk size
- does not install OCS (see docs for details to do this)
- uses latest RHEL cloud access ami images
- install is for ocp 3.11
- uses github oauth application credentials
- new VPC created (can deploy to existing)
- no ASG's
- instructions for letsencrypt and dns setup

Read the docs:
- https://docs.okd.io/latest/install_config/configuring_aws.html
- https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_amazon_web_services
- https://access.redhat.com/documentation/en-us/reference_architectures/2018/html/deploying_and_managing_openshift_3.9_on_amazon_web_services/reference_architecture_summary

### Setup

Use a Bash shell

Install AWS Client on your linux laptop
 
```
pip install awscli --upgrade --user

See: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
```

Configure aws client, you need your `Access Key ID`, `Secret Access Key`

```
aws configure
```

Check creds work

```bash
aws sts get-caller-identity
```

Get the RedHat Cloud Access AMI's (takes a couple business days)

```
# cloud access
https://access.redhat.com/articles/2962171
https://www.redhat.com/en/technologies/cloud-computing/cloud-access
```

Find RHEL private AMI's

```bash
# AWS Console > Resource Groups > AMI's > Private images filter
# Owner : 309956199498

aws ec2 describe-images --owners 309956199498 \
--filters "Name=is-public,Values=false" \
"Name=name,Values=RHEL*7.6*GA*Access*" \
--region ap-southeast-2
```

### Install AWS Infrastructure

From your laptop, set some environment variables, change to suit, small sized vms just to test the install out

```bash
export clusterid="ocp"
export dns_domain="eformat.nz"
export region="ap-southeast-2"
export cidrvpc="172.16.0.0/16"
export cidrsubnets_public=("172.16.0.0/24" "172.16.1.0/24" "172.16.2.0/24")
export cidrsubnets_private=("172.16.16.0/20" "172.16.32.0/20" "172.16.48.0/20")
export ec2_type_bastion="t2.small"
export ec2_type_master="t2.medium"
export ec2_type_infra="t2.large"
export ec2_type_node="t2.large"
export rhel_release="rhel-7.6"
export ec2_type_bastion_ebs_vol_size=25
export ec2_type_master_ebs_vol_size=50
export ec2_type_infra_ebs_vol_size=100
export ec2_type_node_ebs_vol_size=100
export ec2_type_master_number=3
export ec2_type_infra_number=3
export ec2_type_node_number=3
```

Create a public private ssh keypair to be used with ssh-agent and ssh authentication on AWS EC2s

```bash
if [ ! -f ${HOME}/.ssh/${clusterid}.${dns_domain} ]; then
echo 'Enter ssh key password'
while IFS='' read -r passphrase || [[ -n "$passphrase" ]]; do
  ssh-keygen -P ${passphrase} -o -t rsa -f ~/.ssh/${clusterid}.${dns_domain}
  break
done
fi

export sshkey=$(cat ~/.ssh/${clusterid}.${dns_domain}.pub)
```

Get your az's

```bash
export az=($(aws ec2 describe-availability-zones --region=${region} | jq -r '.[][].ZoneName' | sed -e "s/ $//g" | tr '\n' ' '))
echo ${az[@]}
```

Get your ami

```bash
export ec2ami=($(aws ec2 describe-images \
--region ${region} --owners 309956199498 --filters "Name=is-public,Values=false" | \
jq -r '.Images[] | [.Name,.ImageId] | @csv' | \
sed -e 's/,/ /g' | \
sed -e 's/"//g' | \
grep HVM_GA | \
grep Access2-GP2 | \
grep -i ${rhel_release} | \
sort | \
tail -1))
echo ${ec2ami[@]}
```

Deploy IAM users and sleep for 15 sec so that AWS can instantiate the users

```bash
export iamuser=$(aws iam create-user --user-name ${clusterid}.${dns_domain}-admin)
export s3user=$(aws iam create-user --user-name ${clusterid}.${dns_domain}-registry)
sleep 15
```

Create access key for IAM users

```bash
export iamuser_accesskey=$(aws iam create-access-key --user-name ${clusterid}.${dns_domain}-admin)
export s3user_accesskey=$(aws iam create-access-key --user-name ${clusterid}.${dns_domain}-registry)
```

Create and attach policies to IAM users

```bash
cat << EOF > ~/.iamuser_policy_cpk
{
"Version": "2012-10-17",
"Statement": [
{
"Action": [
"ec2:DescribeVolumes",
"ec2:CreateVolume",
"ec2:CreateTags",
"ec2:DescribeInstances",
"ec2:AttachVolume",
"ec2:DetachVolume",
"ec2:DeleteVolume",
"ec2:DescribeSubnets",
"ec2:CreateSecurityGroup",
"ec2:DescribeSecurityGroups",
"ec2:DescribeRouteTables",
"ec2:AuthorizeSecurityGroupIngress",
"ec2:RevokeSecurityGroupIngress",
"elasticloadbalancing:DescribeTags",
"elasticloadbalancing:CreateLoadBalancerListeners",
"elasticloadbalancing:ConfigureHealthCheck",
"elasticloadbalancing:DeleteLoadBalancerListeners",
"elasticloadbalancing:RegisterInstancesWithLoadBalancer",
"elasticloadbalancing:DescribeLoadBalancers",
"elasticloadbalancing:CreateLoadBalancer",
"elasticloadbalancing:DeleteLoadBalancer",
"elasticloadbalancing:ModifyLoadBalancerAttributes",
"elasticloadbalancing:DescribeLoadBalancerAttributes"
],
"Resource": "*",
"Effect": "Allow",
"Sid": "1"
}
]
}
EOF

aws iam put-user-policy \
--user-name ${clusterid}.${dns_domain}-admin \
--policy-name Admin \
--policy-document file://~/.iamuser_policy_cpk

cat << EOF > ~/.iamuser_policy_s3
{
"Version": "2012-10-17",
"Statement": [
{
"Action": [
"s3:*"
],
"Resource": [
"arn:aws:s3:::${clusterid}.${dns_domain}-registry",
"arn:aws:s3:::${clusterid}.${dns_domain}-registry/*"
],
"Effect": "Allow",
"Sid": "1"
}
]
}
EOF

aws iam put-user-policy \
--user-name ${clusterid}.${dns_domain}-registry \
--policy-name S3 \
--policy-document file://~/.iamuser_policy_s3
```

Deploy EC2 keypair

```bash
export keypair=$(aws ec2 import-key-pair \
--key-name ${clusterid}.${dns_domain} \
--region ${region} \
--public-key-material file://~/.ssh/${clusterid}.${dns_domain}.pub)
```

Deploy S3 bucket and policy

```bash
export aws_s3bucket=$(aws s3api create-bucket \
--create-bucket-configuration LocationConstraint=${region} \
--bucket $(echo ${clusterid}.${dns_domain}-registry) \
--region ${region})

aws s3api put-bucket-tagging \
--bucket $(echo ${clusterid}.${dns_domain}-registry) \
--tagging "TagSet=[{Key=Clusterid,Value=${clusterid}}]"

cat << EOF > ~/.s3_policy
{
             "Statement": [
                {
                   "Effect": "Allow",
                   "Principal": {
                      "AWS": "$(echo ${s3user} | jq -r '.User.Arn')"
                   },
                   "Action": "s3:*",
                   "Resource": "arn:aws:s3:::$(echo $aws_s3bucket | jq -r '.Location' | sed -e 's/^http:\/\///g' -e 's/\(.s3.amazonaws.com\)\/*$//g')"
                }
             ]
}
EOF

aws s3api put-bucket-policy \
--bucket $(echo ${clusterid}.${dns_domain}-registry) \
--policy file://~/.s3_policy
```

Deploy VPC and DHCP server

```bash
export vpc=$(aws ec2 create-vpc --region=${region} --cidr-block ${cidrvpc} | jq -r '.')

if [ $region == "us-east-1" ]; then
  export vpcdhcpopts_dnsdomain="ec2.internal"
else
  export vpcdhcpopts_dnsdomain="${region}.compute.internal"
fi

export vpcdhcpopts=$(aws ec2 create-dhcp-options \
--region=${region} \
--dhcp-configuration " \
[ \
{ \"Key\": \"domain-name\", \"Values\": [
\"${vpcdhcpopts_dnsdomain}\" ] }, \
{ \"Key\": \"domain-name-servers\", \"Values\": [
\"AmazonProvidedDNS\" ] } \
]")

aws ec2 modify-vpc-attribute \
--region=${region} \
--enable-dns-hostnames \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId')

aws ec2 modify-vpc-attribute \
--region=${region} \
--enable-dns-support \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId')

aws ec2 associate-dhcp-options \
--region=${region} \
--dhcp-options-id $(echo ${vpcdhcpopts} | jq -r '.DhcpOptions.DhcpOptionsId') \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId')
```

Deploy IGW and attach to VPC

```bash
export igw=$(aws ec2 create-internet-gateway --region=${region})

aws ec2 attach-internet-gateway \
--region=${region} \
--internet-gateway-id $(echo ${igw} | jq -r '.InternetGateway.InternetGatewayId') \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId')
```

Deploy subnets

```bash
for i in {1..3}; do
export subnet${i}_public="$(aws ec2 create-subnet \
--region=${region} \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId') \
--cidr-block ${cidrsubnets_public[$(expr $i - 1 )]} \
--availability-zone ${az[$(expr $i - 1)]}
)"
done

for i in {1..3}; do
export subnet${i}_private="$(aws ec2 create-subnet \
--region=${region} \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId') \
--cidr-block ${cidrsubnets_private[$(expr $i - 1 )]} \
--availability-zone ${az[$(expr $i - 1)]}
)"
done
```

Deploy EIPs

```bash
for i in {0..3}; do
  export eip${i}="$(aws ec2 allocate-address --domain vpc --region=${region})"
done
```

Deploy NatGW’s

```bash
for i in {1..3}; do
j="eip${i}"
k="subnet${i}_public"
export natgw${i}="$(aws ec2 create-nat-gateway \
--region=${region} \
--subnet-id $(echo ${!k} | jq -r '.Subnet.SubnetId') \
--allocation-id $(echo ${!j} | jq -r '.AllocationId') \
)"
done
```

Deploy RouteTables and routes

```bash
export routetable0=$(aws ec2 create-route-table \
--region=${region} \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId'))

aws ec2 create-route \
--route-table-id $(echo ${routetable0} | jq -r '.RouteTable.RouteTableId') \
--destination-cidr-block 0.0.0.0/0 \
--nat-gateway-id $(echo ${igw} | jq -r '.InternetGateway.InternetGatewayId') \
--region=${region} \
> /dev/null 2>&1

for i in {1..3}; do
j="subnet${i}_public"
aws ec2 associate-route-table \
--route-table-id $(echo ${routetable0} | jq -r '.RouteTable.RouteTableId') \
--subnet-id $(echo ${!j} | jq -r '.Subnet.SubnetId') \
--region=${region}
done

for i in {1..3}; do
export routetable${i}="$(aws ec2 create-route-table \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId') \
--region=${region} \
)"
done

for i in {1..3}; do
j="routetable${i}"
k="natgw${i}"
aws ec2 create-route \
--route-table-id $(echo ${!j} | jq -r '.RouteTable.RouteTableId') \
--destination-cidr-block 0.0.0.0/0 \
--nat-gateway-id $(echo ${!k} | jq -r '.NatGateway.NatGatewayId') \
--region=${region} \
> /dev/null 2>&1
done

for i in {1..3}; do
j="routetable${i}"
k="subnet${i}_private"
aws ec2 associate-route-table \
--route-table-id $(echo ${!j} | jq -r '.RouteTable.RouteTableId') \
--subnet-id $(echo ${!k} | jq -r '.Subnet.SubnetId') \
--region=${region}
done
```

Deploy SecurityGroups and rules

```bash
for i in bastion infra master node; do
export awssg_$(echo ${i} | tr A-Z a-z)="$(aws ec2 create-security-group \
--vpc-id $(echo ${vpc} | jq -r '.Vpc.VpcId') \
--group-name ${i} \
--region=${region} \
--description ${i})"
done

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_bastion} | jq -r '.GroupId') \
--ip-permissions '[{"IpProtocol": "icmp", "FromPort": 8, "ToPort": -1,"IpRanges": [{"CidrIp": "0.0.0.0/0"}]},{"IpProtocol": "tcp", "FromPort": 22, "ToPort": 22,"IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_infra} | jq -r '.GroupId') \
--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 80, "ToPort": 80,"IpRanges": [{"CidrIp": "0.0.0.0/0"}]},{"IpProtocol": "tcp", "FromPort": 443, "ToPort":443, "IpRanges": [{"CidrIp": "0.0.0.0/0"}]},{"IpProtocol": "tcp", "FromPort": 9200, "ToPort":9200, "IpRanges": [{"CidrIp": "0.0.0.0/0"}]},{"IpProtocol": "tcp", "FromPort": 9300, "ToPort":9300, "IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_master} | jq -r '.GroupId') \
--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 443, "ToPort":443, "IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'

for i in 2379-2380 24224; do
aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_master} | jq -r '.GroupId') \
--protocol tcp \
--port $i \
--source-group $(echo ${awssg_master} | jq -r '.GroupId')
done

for i in 2379-2380; do
aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_master} | jq -r '.GroupId') \
--protocol tcp \
--port $i \
--source-group $(echo ${awssg_node} | jq -r '.GroupId')
done

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_master} | jq -r '.GroupId') \
--protocol udp \
--port 24224 \
--source-group $(echo ${awssg_master} | jq -r '.GroupId')

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_node} | jq -r '.GroupId') \
--ip-permissions '[{"IpProtocol": "icmp", "FromPort": 8, "ToPort": -1,"IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'

aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_node} | jq -r '.GroupId') \
--protocol tcp \
--port 22 \
--source-group $(echo ${awssg_bastion} | jq -r '.GroupId')

for i in 53 2049 8053 10250 9100 8444; do
aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_node} | jq -r '.GroupId') \
--protocol tcp \
--port $i \
--source-group $(echo ${awssg_node} | jq -r '.GroupId')
done

for i in 53 4789 8053; do
aws ec2 authorize-security-group-ingress \
--region=${region} \
--group-id $(echo ${awssg_node} | jq -r '.GroupId') \
--protocol udp \
--port $i \
--source-group $(echo ${awssg_node} | jq -r '.GroupId')
done
```

Deploy ELB’s

```bash
export elb_masterext=$(aws elb create-load-balancer \
--region=${region} \
--load-balancer-name ${clusterid}-public-master \
--subnets \
$(echo ${subnet1_public} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet2_public} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet3_public} | jq -r '.Subnet.SubnetId') \
--listener Protocol=TCP,LoadBalancerPort=443,InstanceProtocol=TCP,InstancePort=443 \
--security-groups $(echo ${awssg_master} | jq -r '.GroupId') \
--scheme internet-facing \
--tags Key=name,Value=${clusterid}-public-master Key=Clusterid,Value=${clusterid})

aws elb modify-load-balancer-attributes \
--region=${region} \
--load-balancer-name ${clusterid}-public-master \
--load-balancer-attributes "{\"CrossZoneLoadBalancing\":{\"Enabled\":true},\"ConnectionDraining\":{\"Enabled\":false}}"

aws elb configure-health-check \
--region=${region} \
--load-balancer-name ${clusterid}-public-master \
--health-check Target=HTTPS:443/api,HealthyThreshold=3,Interval=5,Timeout=2,UnhealthyThreshold=2

export elb_masterint=$(aws elb create-load-balancer \
--region=${region} \
--load-balancer-name ${clusterid}-private-master \
--subnets \
$(echo ${subnet1_private} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet2_private} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet3_private} | jq -r '.Subnet.SubnetId') \
--listener Protocol=TCP,LoadBalancerPort=443,InstanceProtocol=TCP,InstancePort=443 \
--security-groups $(echo ${awssg_master} | jq -r '.GroupId') \
--scheme internal \
--tags Key=name,Value=${clusterid}-private-master Key=Clusterid,Value=${clusterid})

aws elb modify-load-balancer-attributes \
--region=${region} \
--load-balancer-name ${clusterid}-private-master \
--load-balancer-attributes "{\"CrossZoneLoadBalancing\":{\"Enabled\":true},\"ConnectionDraining\":{\"Enabled\":false}}"

aws elb configure-health-check \
--region=${region} \
--load-balancer-name ${clusterid}-private-master \
--health-check Target=HTTPS:443/api,HealthyThreshold=3,Interval=5,Timeout=2,UnhealthyThreshold=2

export elb_infraext=$(aws elb create-load-balancer \
--region=${region} \
--load-balancer-name ${clusterid}-public-infra \
--subnets \
$(echo ${subnet1_public} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet2_public} | jq -r '.Subnet.SubnetId') \
$(echo ${subnet3_public} | jq -r '.Subnet.SubnetId') \
--listener \
Protocol=TCP,LoadBalancerPort=80,InstanceProtocol=TCP,InstancePort=80 \
Protocol=TCP,LoadBalancerPort=443,InstanceProtocol=TCP,InstancePort=443 \
--security-groups $(echo ${awssg_infra} | jq -r '.GroupId') \
--scheme internet-facing \
--tags Key=name,Value=${clusterid}-public-infra Key=Clusterid,Value=${clusterid})

aws elb modify-load-balancer-attributes \
--region=${region} \
--load-balancer-name ${clusterid}-public-infra \
--load-balancer-attributes "{\"CrossZoneLoadBalancing\":{\"Enabled\":true},\"ConnectionDraining\":{\"Enabled\":false}}"

aws elb configure-health-check \
--region=${region} \
--load-balancer-name ${clusterid}-public-infra \
--health-check Target=TCP:443,HealthyThreshold=2,Interval=5,Timeout=2,UnhealthyThreshold=2
```

Deploy Route53 zones and resources

```bash
export route53_extzone=$(aws route53 create-hosted-zone \
--region=${region} \
--caller-reference $(date +%s) \
--name ${clusterid}.${dns_domain} \
--hosted-zone-config "PrivateZone=False")

export route53_intzone=$(aws route53 create-hosted-zone \
--region=${region} \
--caller-reference $(date +%s) \
--name ${clusterid}.${dns_domain} \
--vpc "VPCRegion=${region},VPCId=$(echo ${vpc} | jq -r '.Vpc.VpcId')" \
--hosted-zone-config "PrivateZone=True")

cat << EOF > ~/.route53_policy
{
            "Changes": [
              {
                "Action": "CREATE",
                "ResourceRecordSet": {
                  "Name": "master.${clusterid}.${dns_domain}",
                  "Type": "CNAME",
                  "TTL": 300,
                  "ResourceRecords": [
                    {
                      "Value": "$(echo ${elb_masterext} | jq -r '.DNSName')"
                    }
                  ]
                }
              }
            ]
          }
EOF

aws route53 change-resource-record-sets \
--region=${region} \
--hosted-zone-id $(echo ${route53_extzone} | jq -r '.HostedZone.Id' | sed 's/\/hostedzone\///g') \
--change-batch file://~/.route53_policy

cat << EOF > ~/.route53_policy2
{
            "Changes": [
              {
                "Action": "CREATE",
                "ResourceRecordSet": {
                  "Name": "master.${clusterid}.${dns_domain}",
                  "Type": "CNAME",
                  "TTL": 300,
                  "ResourceRecords": [
                    {
                      "Value": "$(echo ${elb_masterint} | jq -r '.DNSName')"
                    }
                  ]
                }
              }
            ]
          }
EOF

aws route53 change-resource-record-sets \
--region=${region} \
--hosted-zone-id $(echo ${route53_intzone} | jq -r '.HostedZone.Id' |sed 's/\/hostedzone\///g') \
--change-batch file://~/.route53_policy2

cat << EOF > ~/.route53_policy3
{
            "Changes": [
              {
                "Action": "CREATE",
                "ResourceRecordSet": {
                  "Name": "*.apps.${clusterid}.${dns_domain}",
                  "Type": "CNAME",
                  "TTL": 300,
                  "ResourceRecords": [
                    {
                      "Value": "$(echo ${elb_infraext} | jq -r '.DNSName')"
                    }
                  ]
                }
              }
            ]
          }
EOF

aws route53 change-resource-record-sets \
--region=${region} \
--hosted-zone-id $(echo ${route53_extzone} | jq -r '.HostedZone.Id' | sed 's/\/hostedzone\///g') \
--change-batch file://~/.route53_policy3

cat << EOF > ~/.route53_policy4
{
            "Changes": [
              {
                "Action": "CREATE",
                "ResourceRecordSet": {
                  "Name": "*.apps.${clusterid}.${dns_domain}",
                  "Type": "CNAME",
                  "TTL": 300,
                  "ResourceRecords": [
                    {
                      "Value": "$(echo ${elb_infraext} | jq -r '.DNSName')"
                    }
                  ]
                }
              }
            ]
          }
EOF

aws route53 change-resource-record-sets \
--region=${region} \
--hosted-zone-id $(echo ${route53_intzone} | jq -r '.HostedZone.Id' | sed 's/\/hostedzone\///g') \
--change-batch file://~/.route53_policy4
```

Create EC2 user-data script

```bash
cat << EOF > /tmp/ec2_userdata.sh
#!/bin/bash
if [ "\$#" -ne 2 ]; then exit 2; fi

printf '%s\n' "#cloud-config"

printf '%s' "cloud_config_modules:"
if [ "\$1" == 'bastion' ]; then
  printf '\n%s\n\n' "- package-update-upgrade-install"
else
  printf '\n%s\n\n' "- package-update-upgrade-install
- disk_setup
- mounts
- cc_write_files"
fi

printf '%s' "packages:"
if [ "\$1" == 'bastion' ]; then
  printf '\n%s\n' "- nmap-ncat"
else
  printf '\n%s\n' "- lvm2"
fi

if [ "\$1" != 'bastion' ]; then
  printf '\n%s' 'write_files:
- content: |
    STORAGE_DRIVER=overlay2
    DEVS=/dev/'
  if [[ "\$2" =~ (c5|c5d|i3.metal|m5) ]]; then
    printf '%s' 'nvme1n1'
  else
    printf '%s' 'xvdb'
  fi
  printf '\n%s\n' '    VG=dockervg
    CONTAINER_ROOT_LV_NAME=dockerlv
    CONTAINER_ROOT_LV_MOUNT_PATH=/var/lib/docker
    CONTAINER_ROOT_LV_SIZE=100%FREE
  path: "/etc/sysconfig/docker-storage-setup"
  permissions: "0644"
  owner: "root"'

  printf '\n%s' 'fs_setup:'
  printf '\n%s' '- label: ocp_emptydir
  filesystem: xfs
  device: /dev/'
  if [[ "\$2" =~ (c5|c5d|i3.metal|m5) ]]; then
    printf '%s\n' 'nvme2n1'
  else
    printf '%s\n' 'xvdc'
  fi
  printf '%s' '  partition: auto'
  if [ "\$1" == 'master' ]; then
    printf '\n%s' '- label: etcd
  filesystem: xfs
  device: /dev/'
    if [[ "\$2" =~ (c5|c5d|i3.metal|m5) ]]; then
      printf '%s\n' 'nvme3n1'
    else
      printf '%s\n' 'xvdd'
    fi
    printf '%s' '  partition: auto'
  fi
  printf '\n'

  printf '\n%s' 'mounts:'
  printf '\n%s' '- [ "LABEL=ocp_emptydir", "/var/lib/origin/openshift.local.volumes", xfs, "defaults,gquota" ]'
  if [ "\$1" == 'master' ]; then
    printf '\n%s' '- [ "LABEL=etcd", "/var/lib/etcd", xfs, "defaults,gquota" ]'
  fi
  printf '\n'
fi
EOF

chmod u+x /tmp/ec2_userdata.sh
```

Deploy the bastion EC2

```bash
export ec2_bastion=$(aws ec2 run-instances \
    --region=${region} \
    --image-id ${ec2ami[1]} \
    --count 1 \
    --instance-type ${ec2_type_bastion} \
    --key-name ${clusterid}.${dns_domain} \
    --security-group-ids $(echo ${awssg_bastion} | jq -r '.GroupId') \
    --subnet-id $(echo ${subnet1_public} | jq -r '.Subnet.SubnetId') \
    --associate-public-ip-address \
    --block-device-mappings "DeviceName=/dev/sda1,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_bastion_ebs_vol_size}}" \
    --user-data "$(/tmp/ec2_userdata.sh bastion ${ec2_type_bastion})" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=bastion},{Key=Clusterid,Value=${clusterid}}]")

sleep 30

aws ec2 associate-address \
--region=${region} \
--allocation-id $(echo ${eip0} | jq -r '.AllocationId') \
--instance-id $(echo ${ec2_bastion} | jq -r '.Instances[].InstanceId')
```

Create the master, infra and node EC2s along with EBS volumes

```bash
for i in $( seq 1 ${ec2_type_master_number} ); do
    j="subnet${i}_private"
    export ec2_master${i}="$(aws ec2 run-instances \
        --region=${region} \
        --image-id ${ec2ami[1]} \
        --count 1 \
        --instance-type ${ec2_type_master} \
        --key-name ${clusterid}.${dns_domain} \
        --security-group-ids $(echo ${awssg_master} | jq -r '.GroupId') $(echo ${awssg_node} | jq -r '.GroupId') \
        --subnet-id $(echo ${!j} | jq -r '.Subnet.SubnetId') \
        --block-device-mappings \
            "DeviceName=/dev/sda1,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_master_ebs_vol_size}}" \
            "DeviceName=/dev/xvdb,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_master_ebs_vol_size}}" \
            "DeviceName=/dev/xvdc,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_master_ebs_vol_size}}" \
            "DeviceName=/dev/xvdd,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_master_ebs_vol_size}}" \
        --user-data "$(/tmp/ec2_userdata.sh master ${ec2_type_master})" \
        --tag-specifications "ResourceType=instance,Tags=[ \
            {Key=Name,Value=master${i}}, \
            {Key=Clusterid,Value=${clusterid}}, \
            {Key=ami,Value=${ec2ami}}, \
            {Key=kubernetes.io/cluster/${clusterid},Value=${clusterid}}]"
        )"
done

for i in $( seq 1 ${ec2_type_infra_number} ); do
    j="subnet${i}_private"
    export ec2_infra${i}="$(aws ec2 run-instances \
        --region=${region} \
        --image-id ${ec2ami[1]} \
        --count 1 \
        --instance-type ${ec2_type_infra} \
        --key-name ${clusterid}.${dns_domain} \
        --security-group-ids $(echo ${awssg_infra} | jq -r '.GroupId') $(echo ${awssg_node} | jq -r '.GroupId') \
        --subnet-id $(echo ${!j} | jq -r '.Subnet.SubnetId') \
        --block-device-mappings \
            "DeviceName=/dev/sda1,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_infra_ebs_vol_size}}" \
            "DeviceName=/dev/xvdb,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_infra_ebs_vol_size}}" \
            "DeviceName=/dev/xvdc,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_infra_ebs_vol_size}}" \
            "DeviceName=/dev/xvdd,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_infra_ebs_vol_size}}" \
        --user-data "$(/tmp/ec2_userdata.sh infra ${ec2_type_infra})" \
        --tag-specifications "ResourceType=instance,Tags=[ \
            {Key=Name,Value=infra${i}}, \
            {Key=Clusterid,Value=${clusterid}}, \
            {Key=ami,Value=${ec2ami}}, \
            {Key=kubernetes.io/cluster/${clusterid},Value=${clusterid}}]"
        )"
done

for i in $( seq 1 ${ec2_type_node_number} ); do
    j="subnet${i}_private"
    export ec2_node${i}="$(aws ec2 run-instances \
        --region=${region} \
        --image-id ${ec2ami[1]} \
        --count 1 \
        --instance-type ${ec2_type_node} \
        --key-name ${clusterid}.${dns_domain} \
        --security-group-ids $(echo ${awssg_node} | jq -r '.GroupId') \
        --subnet-id $(echo ${!j} | jq -r '.Subnet.SubnetId') \
        --block-device-mappings \
            "DeviceName=/dev/sda1,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_node_ebs_vol_size}}" \
            "DeviceName=/dev/xvdb,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_node_ebs_vol_size}}" \
            "DeviceName=/dev/xvdc,Ebs={DeleteOnTermination=False,VolumeSize=${ec2_type_node_ebs_vol_size}}" \
        --user-data "$(/tmp/ec2_userdata.sh node ${ec2_type_node})" \
        --tag-specifications "ResourceType=instance,Tags=[ \
            {Key=Name,Value=node${i}}, \
            {Key=Clusterid,Value=${clusterid}}, \
            {Key=ami,Value=${ec2ami}}, \
            {Key=kubernetes.io/cluster/${clusterid},Value=${clusterid}}]"
        )"
done

sleep 300 # wait till instances are in running state
```

Register EC2s to ELB’s

```bash
export elb_masterextreg=$(aws elb register-instances-with-load-balancer \
    --region=${region} \
    --load-balancer-name ${clusterid}-public-master \
    --instances \
      $(echo ${ec2_master1} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_master2} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_master3} | jq -r '.Instances[].InstanceId'))

export elb_masterintreg=$(aws elb register-instances-with-load-balancer \
    --region=${region} \
    --load-balancer-name ${clusterid}-private-master \
    --instances \
      $(echo ${ec2_master1} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_master2} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_master3} | jq -r '.Instances[].InstanceId'))

export elb_infrareg=$(aws elb register-instances-with-load-balancer \
    --region=${region} \
    --load-balancer-name ${clusterid}-public-infra \
    --instances \
      $(echo ${ec2_infra1} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_infra2} | jq -r '.Instances[].InstanceId') \
      $(echo ${ec2_infra3} | jq -r '.Instances[].InstanceId'))
```

Create tags on AWS components

```bash
aws ec2 create-tags --region=${region} --resources $(echo $vpc | jq -r ".Vpc.VpcId") --tags Key=Name,Value=${clusterid}; \
aws ec2 create-tags --region=${region} --resources $(echo ${eip0} | jq -r '.AllocationId') --tags Key=Name,Value=bastion; \
aws ec2 create-tags --region=${region} --resources $(echo ${eip1} | jq -r '.AllocationId') --tags Key=Name,Value=${az[0]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${eip2} | jq -r '.AllocationId') --tags Key=Name,Value=${az[1]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${eip3} | jq -r '.AllocationId') --tags Key=Name,Value=${az[2]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${natgw1} | jq -r '.NatGateway.NatGatewayId') --tags Key=Name,Value=${az[0]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${natgw2} | jq -r '.NatGateway.NatGatewayId') --tags Key=Name,Value=${az[1]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${natgw3} | jq -r '.NatGateway.NatGatewayId') --tags Key=Name,Value=${az[2]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${routetable0} | jq -r '.RouteTable.RouteTableId') --tags Key=Name,Value=routing; \
aws ec2 create-tags --region=${region} --resources $(echo ${routetable1} | jq -r '.RouteTable.RouteTableId') --tags Key=Name,Value=${az[0]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${routetable2} | jq -r '.RouteTable.RouteTableId') --tags Key=Name,Value=${az[1]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${routetable3} | jq -r '.RouteTable.RouteTableId') --tags Key=Name,Value=${az[2]}; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_bastion} | jq -r '.GroupId') --tags Key=Name,Value=Bastion; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_bastion} | jq -r '.GroupId') --tags Key=clusterid,Value=${clusterid}; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_master} | jq -r '.GroupId') --tags Key=Name,Value=Master; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_master} | jq -r '.GroupId') --tags Key=clusterid,Value=${clusterid}; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_infra} | jq -r '.GroupId') --tags Key=Name,Value=Infra; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_infra} | jq -r '.GroupId') --tags Key=clusterid,Value=${clusterid}; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_node} | jq -r '.GroupId') --tags Key=Name,Value=Node; \
aws ec2 create-tags --region=${region} --resources $(echo ${awssg_node} | jq -r '.GroupId') --tags Key=clusterid,Value=${clusterid}
```

Create Config Files for later use in ansible hosts file

```bash
cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}
#<!-- BEGIN OUTPUT -->
Host bastion
  HostName                 $(echo ${eip0} | jq -r '.PublicIp')
  User                     ec2-user
  StrictHostKeyChecking    no
  ProxyCommand             none
  CheckHostIP              no
  ForwardAgent             yes
  ServerAliveInterval      15
  TCPKeepAlive             yes
  ControlMaster            auto
  ControlPath              ~/.ssh/mux-%r@%h:%p
  ControlPersist           15m
  ServerAliveInterval      30
  IdentityFile             ~/.ssh/${clusterid}.${dns_domain}

Host *.compute-1.amazonaws.com
  ProxyCommand             ssh -w 300 -W %h:%p bastion
  user                     ec2-user
  StrictHostKeyChecking    no
  CheckHostIP              no
  ServerAliveInterval      30
  IdentityFile             ~/.ssh/${clusterid}.${dns_domain}

Host *.ec2.internal
  ProxyCommand             ssh -w 300 -W %h:%p bastion
  user                     ec2-user
  StrictHostKeyChecking    no
  CheckHostIP              no
  ServerAliveInterval      30
  IdentityFile             ~/.ssh/${clusterid}.${dns_domain}
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-domaindelegation
#<!-- BEGIN OUTPUT -->
$(echo ${route53_extzone} | jq -r '.DelegationSet.NameServers[]')
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-cpkuser_access_key
#<!-- BEGIN OUTPUT -->
openshift_cloudprovider_aws_access_key=$(echo ${iamuser_accesskey} | jq -r '.AccessKey.AccessKeyId')
openshift_cloudprovider_aws_secret_key=$(echo ${iamuser_accesskey} | jq -r '.AccessKey.SecretAccessKey')
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-cpk
#<!-- BEGIN OUTPUT -->
openshift_cloudprovider_kind=aws
openshift_clusterid=${clusterid}
EOF
cat ~/.ssh/config-${clusterid}.${dns_domain}-cpkuser_access_key | \
    grep -v 'OUTPUT -->' >> \
    ~/.ssh/config-${clusterid}.${dns_domain}-cpk
cat << EOF >> ~/.ssh/config-${clusterid}.${dns_domain}-cpk
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-s3user_access_key
#<!-- BEGIN OUTPUT -->
openshift_hosted_registry_storage_s3_accesskey=$(echo ${s3user_accesskey} | jq -r '.AccessKey.AccessKeyId')
openshift_hosted_registry_storage_s3_secretkey=$(echo ${s3user_accesskey} | jq -r '.AccessKey.SecretAccessKey')
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-s3
#<!-- BEGIN OUTPUT -->
openshift_hosted_manage_registry=true
openshift_hosted_registry_storage_kind=object
openshift_hosted_registry_storage_provider=s3
EOF
cat ~/.ssh/config-${clusterid}.${dns_domain}-s3user_access_key | \
    grep -v 'OUTPUT -->' >> \
    ~/.ssh/config-${clusterid}.${dns_domain}-s3
cat << EOF >> ~/.ssh/config-${clusterid}.${dns_domain}-s3
openshift_hosted_registry_storage_s3_bucket=${clusterid}.${dns_domain}-registry
openshift_hosted_registry_storage_s3_region=${region}
openshift_hosted_registry_storage_s3_chunksize=26214400
openshift_hosted_registry_storage_s3_rootdirectory=/registry
openshift_hosted_registry_pullthrough=true
openshift_hosted_registry_acceptschema2=true
openshift_hosted_registry_enforcequota=true
openshift_hosted_registry_replicas=3
openshift_hosted_registry_selector='node-role.kubernetes.io/infra=true'
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-urls
#<!-- BEGIN OUTPUT -->
openshift_master_default_subdomain=apps.${clusterid}.${dns_domain}
openshift_master_cluster_hostname=master.${clusterid}.${dns_domain}
openshift_master_cluster_public_hostname=master.${clusterid}.${dns_domain}
#<!-- END OUTPUT -->
EOF

cat << EOF > ~/.ssh/config-${clusterid}.${dns_domain}-hosts
[masters]
$(echo ${ec2_master1} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-master'
$(echo ${ec2_master2} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-master'
$(echo ${ec2_master3} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-master'

[etcd]

[etcd:children]
masters

[nodes]
$(echo ${ec2_node1} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-compute'
$(echo ${ec2_node2} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-compute'
$(echo ${ec2_node3} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-compute'
$(echo ${ec2_infra1} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-infra'
$(echo ${ec2_infra2} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-infra'
$(echo ${ec2_infra3} | jq -r '.Instances[].PrivateDnsName') openshift_node_group_name='node-config-infra'

[nodes:children]
masters
EOF
```

### (Optional) Lets Encrypt ACME DNS

Create certificates for the cluster. Ensure master host dns resolves OK. 

```
git clone https://github.com/Neilpang/acme.sh.git
cd acme.sh
-- using https://github.com/Neilpang/acme.sh#currently-acmesh-supports AWS Route53
-- see ~/.aws/credentials
egrep 'AWS_ACCESS_KEY_ID|AWS_SECRET_ACCESS_KEY' dnsapi/dns_aws.sh | head -2
AWS_ACCESS_KEY_ID="xxxxxxx"
AWS_SECRET_ACCESS_KEY="xxxxxxx"
-- request wildcard cert
./acme.sh --issue -d master.ocp.eformat.nz -d *.apps.ocp.eformat.nz --dns dns_aws --dnssleep 100
```

`Notes: DNS Delegation and CAA`

You must ensure root dns is delegated properly for acme.sh to succeed:

- if delegating dns from your registry provider to aws, need to ensure root is also delegated
- in this example eformat.nz is deletegated in registry to Route53, set a NS record for ocp.eformat.nz. using Route53 dns servers for ocp.eformat.nz which is a separate delegation and also in Route53
- CAA for BOTH delgations needs a record set that holds `issuewild`

```
CAA
0 issuewild "letsencrypt.org"
```

Copy certs.

```
scp ~/.acme.sh/master.ocp.eformat.nz/master.ocp.eformat.nz.cer bastion:/tmp
scp ~/.acme.sh/master.ocp.eformat.nz/master.ocp.eformat.nz.key bastion:/tmp
scp ~/.acme.sh/master.ocp.eformat.nz/ca.cer bastion:/tmp
# on bastion as root
ssh bastion
sudo su -
cd /tmp
mkdir /etc/ansible
mv master.ocp.eformat.nz.cer /etc/ansible
mv master.ocp.eformat.nz.key /etc/ansible
mv ca.cer /etc/ansible
```

Add to ansible/hosts file (see below for full file) e.g.

```
# master api certs
openshift_master_overwrite_named_certificates=true
openshift_master_named_certificates=[{"certfile": "{{ inventory_dir }}/master.ocp.eformat.nz.cer", "keyfile": "{{ inventory_dir }}/master.ocp.eformat.nz.key", "names": ["master.ocp.eformat.nz"], "cafile": "{{ inventory_dir }}/ca.cer"}]
```

Configure edge router certs once OCP installed

```
cat ~/.acme.sh/master.ocp.eformat.nz/master.ocp.eformat.nz.cer ~/.acme.sh/master.ocp.eformat.nz/master.ocp.eformat.nz.key ~/.acme.sh/master.ocp.eformat.nz/ca.cer > /tmp/cloudapps.router.pem
oc secrets new router-certs tls.crt=/tmp/cloudapps.router.pem tls.key=/home/mike/.acme.sh/master.ocp.eformat.nz/master.ocp.eformat.nz.key -o json --type='kubernetes.io/tls' -n default --confirm | oc replace -n default -f-
```

### Install OpenShift

Prep the bastion host, as ec2-user

```
scp ~/.ssh/config-ocp.eformat.nz bastion:~/.ssh
scp ~/.ssh/ocp.eformat.nz.pub bastion:~/.ssh
scp ~/.ssh/ocp.eformat.nz bastion:~/.ssh

ssh bastion
cd .ssh
chmod 400 config-ocp.eformat.nz
chmod 400 ocp.eformat.nz.pub
chmod 400 ocp.eformat.nz

if [ ! "$(env | grep SSH_AGENT_PID)" ] || [ ! "$(ps -ef | grep -v grep |
grep ${SSH_AGENT_PID})" ]; then
rm -rf ${SSH_AUTH_SOCK} 2> /dev/null
unset SSH_AUTH_SOCK
unset SSH_AGENT_PID
pkill ssh-agent
export sshagent=$(nohup ssh-agent &)
export sshauthsock=$(echo ${sshagent} | awk -F'; ' {'print $1'})
export sshagentpid=$(echo ${sshagent} | awk -F'; ' {'print $3'})
export ${sshauthsock}
export ${sshagentpid}
for i in sshagent sshauthsock sshagentpid; do
unset $i
done
fi

export sshkey=($(cat ~/.ssh/ocp.eformat.nz))
IFS=$'\n'; if [ ! $(ssh-add -L | grep ${sshkey[1]}) ]; then
  ssh-add ~/.ssh/ocp.eformat.nz
fi
unset IFS

ssh-add -L

ln -s config-ocp.eformat.nz config
```

Delegate DNS to your top level domain (eformat.nz) - create public route53, delegate dns to aws

Create NS record in Route53 that points to sub domain nameservers i.e.

```
cat ~/.ssh/config-ocp.eformat.nz-domaindelegation
```

`(Letsencrypt Optional)`

Create CAA for letsencrypt (See above) in Route53.

If running your own bind locally, use NS for top level domain (Route53) so we can resolve CAA
```
-- /etc/named.conf

        // dnsmasq, aws, google, forwarders
        forwarders { 127.0.0.1 port 853; 205.251.193.246; 8.8.8.8; };

dig eformat.nz caa
dig ocp.eformat.nz caa
```

Using github oauth application for login. Create a github org.

```
Settings >  Developer settings > OAuth Apps
New OAuth App > ocp_3.11_aws
https://master.ocp.eformat.nz
https://master.ocp.eformat.nz/oauth2callback/github
```

Disbale rhui manually on bastion

```
sudo su -
yum-config-manager \
--disable 'rhui-REGION-client-config-server-7' \
--disable 'rhui-REGION-rhel-server-rh-common' \
--disable 'rhui-REGION-rhel-server-releases'
```

Subscribe bastion, openshift pool

```
subscription-manager register --username=<user> --password=<pwd>
subscription-manager subscribe --pool=<pool>
```

OCP 3.11

```
subscription-manager repos --disable="*" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"
```

RPM prereqs

```
yum -y install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
yum -y install ansible atomic-openshift-utils atomic-client-utils openshift-ansible
```

Setup a basic ansible/hosts

```
# local machine
scp ~/.ssh/config-ocp.eformat.nz-hosts bastion:/tmp/hosts
# as root on bastion
cp /tmp/hosts /etc/ansible/hosts
```

As ec2-user, from bastion, check ssh, this should succeed all node hosts

```
export ANSIBLE_HOST_KEY_CHECKING=False
ansible nodes -b -m ping
```

Prepare ALL Node hosts

Disbale RHUI

```
ansible nodes -b -m command -a "yum-config-manager \
--disable 'rhui-REGION-client-config-server-7' \
--disable 'rhui-REGION-rhel-server-rh-common' \
--disable 'rhui-REGION-rhel-server-releases'"
```

Subscription manager

```
ansible nodes -b -m redhat_subscription -a \
"state=present username=<user> password=<password> pool_ids=<pool>"
```

Repos

```
ansible nodes -b -m shell -a \
'subscription-manager repos --disable="*" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"'
```

Create a bigger `/etc/ansible/hosts` file on the bastion as root user

Credentials you need:
- RedHat - Create a service account for terms based registry `registry.redhat.io` for authenticated access: https://access.redhat.com/terms-based-registry
- AWS access and secret keys
- Github oauth application based login

```
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd
# lb

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_ssh_user=ec2-user
ansible_become=yes
openshift_deployment_type=openshift-enterprise
openshift_release=v3.11
openshift_image_tag=v3.11.59
openshift_master_dynamic_provisioning_enabled=True

# registry to install from
oreg_auth_user=<terms based registry user>
oreg_auth_password=<terms based registry pwd>
oreg_url=registry.redhat.io/openshift3/ose-${component}:${version}
oreg_auth_credentials_replace=true

# login
openshift_master_identity_providers=[{'name': 'github', 'challenge':'false', 'login': 'true', 'kind': 'GitHubIdentityProvider','mapping_method': 'claim', 'clientID': '<client id>','clientSecret': '<client secret>'}]

# master api certs
openshift_master_overwrite_named_certificates=true
openshift_master_named_certificates=[{"certfile": "{{ inventory_dir }}/master.ocp.eformat.nz.cer", "keyfile": "{{ inventory_dir }}/master.ocp.eformat.nz.key", "names": ["master.ocp.eformat.nz"], "cafile": "{{ inventory_dir }}/ca.cer"}]

# config-ocp.eformat.nz-cpk
openshift_cloudprovider_kind=aws
openshift_clusterid=ocp
openshift_cloudprovider_aws_access_key=<aws access key>
openshift_cloudprovider_aws_secret_key=<aws secret key>

# config-ocp.eformat.nz-s3
openshift_hosted_manage_registry=true
openshift_hosted_registry_storage_kind=object
openshift_hosted_registry_storage_provider=s3
openshift_hosted_registry_storage_s3_accesskey=<aws iam access key>
openshift_hosted_registry_storage_s3_secretkey=<aws iam secret key>
openshift_hosted_registry_storage_s3_bucket=ocp.eformat.nz-registry
openshift_hosted_registry_storage_s3_region=ap-southeast-2
openshift_hosted_registry_storage_s3_chunksize=26214400
openshift_hosted_registry_storage_s3_rootdirectory=/registry
openshift_hosted_registry_pullthrough=true
openshift_hosted_registry_acceptschema2=true
openshift_hosted_registry_enforcequota=true
openshift_hosted_registry_replicas=3
openshift_hosted_registry_selector='node-role.kubernetes.io/infra=true'

# config-ocp.eformat.nz-urls
openshift_master_cluster_method=native
openshift_master_default_subdomain=apps.ocp.eformat.nz
openshift_master_cluster_hostname=master.ocp.eformat.nz
openshift_master_cluster_public_hostname=master.ocp.eformat.nz

# not atomic - if set to false or unset, the default RPM method is used
containerized=false

# Configure the multi-tenant SDN plugin (default is 'redhat/openshift-ovs-subnet')
os_sdn_network_plugin_name='redhat/openshift-ovs-networkpolicy'

# Master API Port
openshift_master_api_port=443
openshift_master_console_port=443

# Examples
openshift_install_examples=true
openshift_examples_load_xpaas=true
openshift_examples_load_quickstarts=true
openshift_examples_load_db_templates=true
openshift_examples_registryurl=registry.redhat.io
openshift_examples_modify_imagestreams=true
openshift_examples_load_centos=false
openshift_examples_load_rhel=true

# Metrics (Hawkular)
openshift_metrics_install_metrics=false
openshift_master_metrics_public_url=https://hawkular-metrics.apps.eformat.nz/hawkular/metrics

# Prometheus
openshift_hosted_prometheus_deploy=false
openshift_prometheus_node_selector='node-role.kubernetes.io/compute=true'

# logging
openshift_logging_install_logging=false
openshift_master_logging_public_url=https://kibana.apps.eformat.nz
logrotate_scripts=[{"name": "syslog", "path": "/var/log/cron\n/var/log/maillog\n/var/log/messages\n/var/log/secure\n/var/log/spooler\n", "options": ["daily", "rotate 7", "compress", "sharedscripts", "missingok"], "scripts": {"postrotate": "/bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true"}}]

# docker log options
openshift_docker_options="--log-driver json-file --log-opt max-size=1M --log-opt max-file=3"

# pre-install checks disable
openshift_disable_check=memory_availability,disk_availability,docker_storage,package_availability,package_version,docker_image_availability

# config-ocp.eformat.nz-hosts
[masters]
ip-172-16-26-107.ap-southeast-2.compute.internal openshift_node_group_name='node-config-master'
ip-172-16-37-85.ap-southeast-2.compute.internal openshift_node_group_name='node-config-master'
ip-172-16-55-231.ap-southeast-2.compute.internal openshift_node_group_name='node-config-master'

[etcd]

[etcd:children]
masters

[nodes]
ip-172-16-28-82.ap-southeast-2.compute.internal openshift_node_group_name='node-config-compute'
ip-172-16-46-127.ap-southeast-2.compute.internal openshift_node_group_name='node-config-compute'
ip-172-16-50-103.ap-southeast-2.compute.internal openshift_node_group_name='node-config-compute'
ip-172-16-24-171.ap-southeast-2.compute.internal openshift_node_group_name='node-config-infra'
ip-172-16-45-245.ap-southeast-2.compute.internal openshift_node_group_name='node-config-infra'
ip-172-16-49-56.ap-southeast-2.compute.internal openshift_node_group_name='node-config-infra'

[nodes:children]
masters
```

Setup Container storage and other deps from bastion as ec2-user

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
```

Deploy OpenShift

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

### Delete it ALL

The AWS bits are notorious for being dependent on each other. AWS EC2,VPC Dashbaord's are the best place to delete everything.

RedHat remove subscriptions from bastion

```
ansible nodes -b -m shell -a 'subscription-manager unsubscribe --all'
ansible nodes -b -m shell -a 'subscription-manager unregister'
```

From your laptop

Environment variables for deletion

```
export region="ap-southeast-2"
export clusterid="ocp"
export dns_domain="eformat.nz"
```

Delete instances
```
aws ec2 terminate-instances --region=${region} --instance-ids $(aws ec2 describe-instances --region=${region} --filters "Name=instance-state-name,Values=pending,running,stopped,stopping" --query "Reservations[].Instances[].[InstanceId]" --output text | tr '\n' ' ')

sleep 100
```

Delete Volumes

```
for i in `aws ec2 describe-volumes --region=${region} --query "Volumes[].VolumeId" --output text | tr '\n' ' '`; do aws ec2 delete-volume --region=${region} --volume-id $i; done
```

Get VpcId

```
export vpcid=$(aws ec2 describe-vpcs --region=${region} --filters Name=tag:Name,Values=${clusterid} --query "Vpcs[].VpcId" | jq '.[0]' | sed -e 's/"//g')
echo ${vpcid}
```

Delete Load Balancers

```
for i in `aws elb describe-load-balancers --region=${region} --query="LoadBalancerDescriptions[].LoadBalancerName" --output text | tr '\n' ' '`; do aws elb delete-load-balancer --region=${region} --load-balancer-name ${i}; done
```

Delete NAT Gatewats

```
for i in `aws ec2 describe-nat-gateways --region=${region} --query="NatGateways[].NatGatewayId" --output text | tr '\n' ' '`; do aws ec2 delete-nat-gateway --nat-gateway-id ${i} --region=${region}; done

sleep 100
```

Delete the VPC

```
aws ec2 delete-vpc --vpc-id ${vpcid} --region=${region}
```

Delete Route Tables

```
for i in `aws ec2 describe-route-tables --region=${region} --query="RouteTables[].RouteTableId" --output text | tr '\n' ' '`; do aws ec2 delete-route-table --region=${region} --route-table-id=${i}; done
```

Delete subnets

```
for i in `aws ec2 describe-subnets --region=${region} --filters Name=vpc-id,Values="${vpcid}" | grep subnet- | sed -E 's/^.*(subnet-[a-z0-9]+).*$/\1/'`; do aws ec2 delete-subnet --region=${region} --subnet-id=$i; done
```

Delete ENI's

```
for i in `aws ec2 describe-network-interfaces --region=${region} --query "NetworkInterfaces[].NetworkInterfaceId" | grep eni- | sed -E 's/^.*(eni-[a-z0-9]+).*$/\1/'`; do aws ec2 delete-network-interface --region=${region} --network-interface-id=${i}; done
```

Delete internet gateways

```
for i in `aws ec2 describe-internet-gateways --region=${region} --filters Name=attachment.vpc-id,Values="${vpcid}" | grep igw- | sed -E 's/^.*(igw-[a-z0-9]+).*$/\1/'`; do aws ec2 delete-internet-gateway --region=${region} --internet-gateway-id=$i; done
```

Delete security groups (ignore message about being unable to delete default security group)

```
for i in `aws ec2 describe-security-groups --region=${region} --filters Name=vpc-id,Values="${vpcid}" | grep sg- | sed -E 's/^.*(sg-[a-z0-9]+).*$/\1/' | sort | uniq`; do aws ec2 delete-security-group --region=${region} --group-id $i; done
```

Delete S3

```
aws s3api delete-bucket --region ${region} --bucket $(aws s3api list-buckets --region ${region} --query "Buckets[].Name" --output text | tr '\n' ' ')
```

Delete Keys

```
aws ec2 delete-key-pair --region ${region} --key-name ${clusterid}.${dns_domain}
```

Delete Users

```
aws iam delete-user --user-name ${clusterid}.${dns_domain}-admin
aws iam delete-user --user-name ${clusterid}.${dns_domain}-registry
```
