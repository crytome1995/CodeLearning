**SETTING UP THE VPC**

1.  Create the vpc  
    This will host our aws ec2 instances

    ```sh
    aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId'
     aws ec2 create-tags --resources vpc-073742bde8f035d7e --tags Key=Name,Value=ethans-vpc
     aws ec2 modify-vpc-attribute --vpc-id vpc-073742bde8f035d7e --enable-dns-support "{\"Value\": true}"
     aws ec2 modify-vpc-attribute --vpc-id vpc-073742bde8f035d7e --enable-dns-hostnames "{\"Value\": true}"
    ```

    VPC_ID = vpc-073742bde8f035d7e

    1.1 first command creates a vpc  
     1.2 second command tags our vpc with a name  
     1.3 enables dns support for the vpc  
     1.4 allows for instances in the vpc to receive dns hostnames

2.  Create the subnets in the vpc  
    This will host our ec2 instances  
    2.1 Create the public subnet /24 prefix 256 hosts
    cidr block: 10.0.1.0/24

    ```
    aws ec2 create-subnet --vpc-id vpc-073742bde8f035d7e --cidr-block 10.0.1.0/24 --output text --query 'Subnet.SubnetId'
    ```

    PUB_SUB_ID = subnet-091e5d9837a98561b

    ```
    aws ec2 create-tags --resources subnet-091e5d9837a98561b --tags Key=Name,Value=kubernetes-Public Key=Type,Value=Public
    ```

    2.2 Create the private subnet /24 prefix 256 hosts

    ```
    aws ec2 create-subnet --vpc-id vpc-073742bde8f035d7e --cidr-block 10.0.2.0/24 --output text --query 'Subnet.SubnetId'
    ```

    PRIV_SUB_ID = subnet-096bbe24749ea96e8

    ```
    aws ec2 create-tags --resources subnet-096bbe24749ea96e8 --tags Key=Name,Value=kubernetes-Private Key=Type,Value=Private
    ```

3.  Create the internet gateway and NAT gateway
    Allows for public subnet to talk to the internet must associate with a route table

    ```
    aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId'
    ```

    IGW_ID = igw-0886b7f7b9cf4d8c7

    ```
    aws ec2 create-tags --resources igw-0886b7f7b9cf4d8c7 --tags Key=Name,Value=kubernetes
    ```

    Attach IGW to VPC

    ```
    aws ec2 attach-internet-gateway --internet-gateway-id igw-0886b7f7b9cf4d8c7 --vpc-id vpc-073742bde8f035d7e
    ```

    Provision an EIP on AWS console, create a NAT and associate the EIP with it, bind the NAT to the public subnet, add all internet bound traffic 0.0.0.0/0 to point to the NAT for the private subnet route table

4.  Create the route table  
    Dictate traffic to your vpc subnets are gateways  
     4.1 Public Route table to allow traffic to ips in public subnet

    ```
    aws ec2 create-route-table --vpc-id vpc-073742bde8f035d7e --output text --query 'RouteTable.RouteTableId'
    ```

    RTB_ID = rtb-04558009fd6851678

    ```
    aws ec2 create-tags --resources rtb-04558009fd6851678 --tags Key=Name,Value=kubernetes-Public Key=Type,Value=Public
    ```

    ```
    aws ec2 associate-route-table --route-table-id rtb-04558009fd6851678 --subnet-id subnet-091e5d9837a98561b
    ```

    Add a route allowing all traffic 0.0.0.0 meaning any packet (default)

    ```
    aws ec2 create-route --route-table-id rtb-04558009fd6851678 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0886b7f7b9cf4d8c7
    ```

    4.2 Create the prive route table with associated NAT gateway

5.  Create the security groups (firewalls) to control acces to EC2 instances  
    5.1 Public security group

    ```
    aws ec2 create-security-group --group-name kubernetes --description "Kubernetes security group public" --vpc-id vpc-073742bde8f035d7e
    ```

    PUB_SG_ID = sg-0e08ae79857416ceb

    ```
    aws ec2 create-tags --resources sg-0e08ae79857416ceb --tags Key=Name,Value=kubernetes-Public Key=Type,Value=Public
    ```

    Add these to the security group:

    ```
    protocol all --cidr 10.0.0.0/16
    protocol all --cidr 10.200.0.0/16
    protocol tcp --port 22 --cidr 0.0.0.0/0
    protocol tcp --port 6443 --cidr 0.0.0.0/0
    protocol tcp --port 443 --cidr 0.0.0.0/0
    protocol icmp --port -1 --cidr 0.0.0.0/0
    ```

6.  Create the Loadbalancer interact with ec2 instances

    ```
    aws elbv2 create-load-balancer --name kubernetes --subnets subnet-091e5d9837a98561b --scheme internet-facing --type network
    ```

    ARN: arn:aws:elasticloadbalancing:us-east-1:982658172500:loadbalancer/net/kubernetes/1d91b99c452dbea0

    #### Setup target group for kube api

    Target groups represent groups of ec2 instances

    ```
    aws elbv2 create-target-group --name kubernetes --protocol TCP --port 6443 --vpc-id vpc-073742bde8f035d7e --target-type ip
    ```

    TG_ARN = arn:aws:elasticloadbalancing:us-east-1:982658172500:targetgroup/kubernetes/655941910c299268

    ```
    aws elbv2 register-targets --target-group-arn arn:aws:elasticloadbalancing:us-east-1:982658172500:targetgroup/kubernetes/655941910c299268 --targets Id=10.0.2.1
    ```

    Register the target group with the loadbalancer, this is how the loadbalancer determines healthy instances

    ELB DNS OUTPUT: kubernetes-1d91b99c452dbea0.elb.us-east-1.amazonaws.com

    ```
    aws elbv2 describe-load-balancers --load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:982658172500:loadbalancer/net/kubernetes/1d91b99c452dbea0
    ```

    aws ec2 describe-images --owners 099720109477 --filters "Name=root-device-type,Values=ebs" "Name=architecture,Values=x86_64" "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-\*" | jq -r '.Images|sort_by(.Name)[-1]|.ImageId'

    ami-0c536cd6abac1a385

7.  Setup our EC2's
    7.1 Create the SSH key pairs (how we can get to the boxes)

    ```
    aws ec2 create-key-pair --key-name bastion --output text --query "KeyMaterial" > bastion.id_rsa
    chmod 600 bastion.id_rsa
    ```

    7.2 Start master nodes

    ```
    aws ec2 run-instances --image-id ami-0c536cd6abac1a385 --count 1 --key-name kubernetes --security-group-ids sg-0e08ae79857416ceb --instance-type t3.micro --private-ip-address 10.0.2.10 --user-data "name=controller-0" --subnet-id subnet-096bbe24749ea96e8 --block-device-mappings="{\"DeviceName\": \"/dev/sda1\", \"Ebs\": { \"VolumeSize\": 50 }, \"NoDevice\": \"\" }" --output text --query "Instances[].InstanceId"
    ```

    ### Disable source dest check, this allows for us to hit the loadbalancer and pass to the ec2 instance

    ```
    aws ec2 modify-instance-attribute --instance-id i-0e099c1117ebab4d3 --no-source-dest-check
    ```

    master0 ID: i-0d5531285b237ae5d  
    master1 ID: i-082fffee815d71350  
    master2 ID: i-0e099c1117ebab4d3

    ```
    aws ec2 create-tags --resources i-0e099c1117ebab4d3 --tags "Key=Name,Value=master-2"
    ```

    7.3 Start the worker nodes

    ```
    aws ec2 run-instances --image-id ami-0c536cd6abac1a385 --count 1 --key-name kubernetes --security-group-ids sg-0e08ae79857416ceb --instance-type t3.micro --private-ip-address 10.0.2.20 --user-data "name=worker-0" --subnet-id subnet-096bbe24749ea96e8 --block-device-mappings="{\"DeviceName\": \"/dev/sda1\", \"Ebs\": { \"VolumeSize\": 50 }, \"NoDevice\": \"\" }" --output text --query "Instances[].InstanceId"
    ```

    ```
    aws ec2 modify-instance-attribute --instance-id i-00ea71c462c31d2dd --no-source-dest-check
    ```

    worker0 ID: i-0c7099db4a95e397b  
    worker1 ID: i-089db788aee8542c3  
    worker2 ID: i-00ea71c462c31d2dd

    ```
    aws ec2 create-tags --resources i-00ea71c462c31d2dd --tags "Key=Name,Value=worker-2"
    ```

    ```
    aws ec2 create-tags --resources i-05480d296c1d102b6 --tags "Key=Name,Value=worker-1"
    ```

8.  Setting up the PKI for tls communication and trust in the cluster

    8.1 Provisiob the CA (Certificate authority) which allows for generating and signing certs

    ```
    {
    "signing": {
        "default": {
        "expiry": "8760h"
        },
        "profiles": {
        "kubernetes": {
            "usages": ["signing", "key encipherment", "server auth", "client auth"],
            "expiry": "8760h"
        }
        }
    }
    }
    ```

    Create the certificate signing request

    ```
    {
        "CN": "Kubernetes",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [{
            "C": "US",
            "L": "Portland",
            "O": "Kubernetes",
            "OU": "CA",
            "ST": "Oregon"
        }]
    }
    ```

    Generate the cert and private key (This is a self-signed)

    ```
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
    ```

    8.2 Create the admin client certificate

    ```
    {
    "CN": "admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "C": "US",
        "L": "Portland",
        "O": "system:masters",
        "OU": "Kubernetes The Hard Way",
        "ST": "Oregon"
        }
    ]
    }
    ```

    Generate the cert and key using the CA as a signer

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
    ```

    Generates admin key and admin cert

    ##### The profile is for assigning certs to groups

    8.3 Kube componenet certs

    Kubelet certs

    ##### Kubelets run on each node and these make api calls to kubernetes which need to be authenticated by a node authorizer

    - Each cert for the kubelet requires the CN to point to the servers host-name

    Get IP of instance

    ```
    aws ec2 describe-instances --filters "Name=tag:Name,Values=worker-1" "Name=instance-state-name,Values=running" --output text --query "Reservations[].Instances[].PublicIpAddress"
    ```

    ```
    aws ec2 describe-instances --filters "Name=tag:Name,Values=master-1" "Name=instance-state-name,Values=running" --output text --query "Reservations[].Instances[].PublicIpAddress"
    ```

    ```
    aws ec2 describe-instances --filters "Name=tag:Name,Values=worker-0" "Name=instance-state-name,Values=running" --output text --query "Reservations[].Instances[].PrivateIpAddress"
    ```

    Worker-0 IP: 34.207.234.105  
    Private: 10.0.2.20  
    Worker-1 IP: 3.87.112.119  
    Private: 10.0.2.21  
    Worker-2 IP: 3.89.100.79  
    Private: 10.0.2.22

    Master-0 IP: 54.209.214.99  
    Private: 10.0.2.10  
    Master-1 IP: 107.23.145.50  
    Master-2 IP: 54.159.30.223

    Generate the cert and key for workers

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -hostname=ip-10-0-2-22,3.89.100.79,10.0.2.22 -profile=kubernetes worker-csr.json | cfssljson -bare worker-2
    ```

    Generate the cert and key for the masters

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -hostname=ip-10-0-2-10,10.0.2.10 -profile=kubernetes master-csr.json | cfssljson -bare master
    ```

    Generate the cert and key for kube-controll-manager: which handles the controll loop of kubernetes to make sure the desired state is met

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
    ```

    Generate the cert for the kube-proxy: which handles the network traffic between nodes

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
    ```

    Generate the kube-scheduler cert that handles scheduling pods to nodes

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
    ```

    Generate the kubernetes API server certificate

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -hostname=10.32.0.1,10.0.2.10,10.0.2.11,10.0.2.12,kubernetes-1d91b99c452dbea0.elb.us-east-1.amazonaws.com,127.0.0.1,kubernetes.default -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
    ```

    Generate the service account controller cert

    ```
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes service-account-csr.json | cfssljson -bare service-account
    ```

    8.2 Copy over certs to worker nodes

    ```
    scp -i kubernetes.id_rsa ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem ubuntu@ec2-3-87-112-119.compute-1.amazonaws.com:~/
    ```

```
for instance in master-0 master-1 master-2; do
    external_ip=$(aws ec2 describe-instances --filters \
        "Name=tag:Name,Values=${instance}" \
        "Name=instance-state-name,Values=running" \
        --output text --query "Reservations[].Instances[].PrivateDnsName")

    scp -i kubernetes.id_rsa kubernetes.pem  kubernetes-key.pem ubuntu@${external_ip}:~/
done
```
