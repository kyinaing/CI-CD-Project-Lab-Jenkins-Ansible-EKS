# CI-CD-Project-Lab-Jenkins-Ansible-EKS

This project note exemplifies a comprehensive DevOps implementation employing GitHub, EKS, Docker Hub, Git, Jenkins, and Ansible. Delve into the seamless integration of these tools to optimize software delivery, automate workflows, and foster collaboration, resulting in accelerated and superior development outcomes.
![image](https://github.com/kyinaing/CI-CD-Project-Lab-Jenkins-Ansible-EKS/assets/12751896/78f7bbf2-170b-4807-9b49-86cb79d8d9dd)


# Install and create required server for Docker-host and Ansible Server

**We will create two server for EKS-Bootstrap-Server and Ansible Server and install Docker and Ansible.

![image](https://github.com/kyinaing/CI-CD-Project-Lab-Jenkins-Ansible-EKS/assets/12751896/b85cda6d-8168-42a4-acef-379475d335a5)


# Create ansadmin user
**We have to create ansadmin user on both docker and ansible server to run ansible playbook and then give sudo permission.

- On EKS-Bootstrap-Server
```
[root@EKS-Bootstrap-server ~]# cat /etc/passwd
ec2-user:x:1000:1000:EC2 Default User:/home/ec2-user:/bin/bash
dockeradmin:x:1001:1001::/home/dockeradmin:/bin/bash
ansadmin:x:1002:1002::/home/ansadmin:/bin/bash
```
- On Ansible-server
```
ansadmin@ansible-server:~$ sudo cat /etc/passwd
ansadmin:x:1001:1001::/home/ansadmin:/bin/bash
```
# Copy SSH Key to Docker-host from ansible-server
ssh-copy-id 172.31.3.248
```
[root@EKS-Bootstrap-server ~]## cat .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDfKXE88+ZMMf52VJsbWTv58xxxxxxxxxxxxxxxx99MxT8KQCh1Mp8= ansadmin@ansible-server
```
# Install Publish over SSH and add SSH server on Jenkins Server
**To send build artifact over SSH to ansible server, the installation of the "Publish over SSH" plug-in on the Jenkins server is required.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/ac6f5f39-8fea-43da-aa8f-c4d2b7481e12)

**And add ansible server with username and password on Jenkins Server from Manage Jenkins and Configure setting from System configuration.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/d3eceab5-f4cb-496c-a3f1-919413c6e0f4)

# Prepare Artifact Location, Dockerfile and Ansible Playbook

**We will push our application artifact in /opt/docker and we will write dockerfile for our application.

```
ansadmin@ansible-server:/opt/docker$ cat Dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```
And we will add hosts for docker-host in ansible hosts file

```
ansadmin@ansible-server:/opt/docker$ cat /etc/ansible/hosts

#This is the default ansible 'hosts' file.

#It should live in /etc/ansible/hosts

#- Comments begin with the '#' character
#- Blank lines are ignored
#- Groups of hosts are delimited by [header] elements
#- You can enter hostnames or ip addresses
#- A hostname/ip can be a member of multiple groups

[ansible]
172.31.24.126
[kubernetes]
172.31.7.117
```
**And we will check our ansible host connection

```
ansadmin@ansible-server:~$ ansible all -a uptime
172.31.24.126 | CHANGED | rc=0 >>
 11:52:47 up 8 min,  2 users,  load average: 0.03, 0.06, 0.05
[WARNING]: Platform linux on host 172.31.7.117 is using the discovered Python interpreter at /usr/bin/python3.9,
but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.14/reference_appendices/interpreter_discovery.html for more information.
172.31.7.117 | CHANGED | rc=0 >>
 11:52:47 up  2:37,  5 users,  load average: 0.07, 0.02, 0.00
 ```
 **Now we will bootstrap EKS kubernetes cluster with eksctl
 
 To create EKS Cluster with eksctl by ec2 bootstrap server, we have to create the following resource,
 - EC2
 - IAM Role with AmazonEC2FullAccess, IAMFullAccess, AWSCloudFormationFullAccess, EKSFullAccess (CustomerManaged), IAMLimitAccess (CustomerManaged)
 - assign IAM Role to Ec2
 - install kubectl
 - install eksctl

EKSFullAccess
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "eks.amazonaws.com"
                }
            }
        }
    ]
}
```
IAMLimitAccess
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateInstanceProfile",
                "iam:DeleteInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:ListInstanceProfiles",
                "iam:AddRoleToInstanceProfile",
                "iam:ListInstanceProfilesForRole",
                "iam:PassRole",
                "iam:DetachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:GetOpenIDConnectProvider",
                "iam:CreateOpenIDConnectProvider",
                "iam:DeleteOpenIDConnectProvider",
                "iam:TagOpenIDConnectProvider",
                "iam:ListAttachedRolePolicies",
                "iam:TagRole",
                "iam:GetPolicy",
                "iam:CreatePolicy",
                "iam:DeletePolicy",
                "iam:ListPolicyVersions"
            ],
            "Resource": [
                "arn:aws:iam::<youraccoundid>:instance-profile/eksctl-*",
                "arn:aws:iam::<youraccoundid>:role/eksctl-*",
                "arn:aws:iam::<youraccoundid>:policy/eksctl-*",
                "arn:aws:iam::<youraccoundid>:oidc-provider/*",
                "arn:aws:iam::<youraccoundid>:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                "arn:aws:iam::<youraccoundid>:role/eksctl-managed-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole"
            ],
            "Resource": [
                "arn:aws:iam::<youraccoundid>:role/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "eks.amazonaws.com",
                        "eks-nodegroup.amazonaws.com",
                        "eks-fargate.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```
![image](https://github.com/kyinaing/CI-CD-Project-Lab-Jenkins-Ansible-EKS/assets/12751896/8c386336-18ab-4bcc-b130-4754d79f8154)
![image](https://github.com/kyinaing/CI-CD-Project-Lab-Jenkins-Ansible-EKS/assets/12751896/1d7d0465-d9a9-479d-8e00-ec9414ec1117)
  ![image](https://github.com/kyinaing/CI-CD-Project-Lab-Jenkins-Ansible-EKS/assets/12751896/5983466c-656c-4733-aa04-bba7a30206c8)

After complete previous steps, we will bootstrap eks cluster with eksctl
**eksctl create cluster --name kube-cluster --region ap-southeast-2 --node-type t2.small

```
[ansadmin@EKS-Bootstrap-server ~]$ eksctl create cluster --name kube-cluster --region ap-southeast-2 --node-type t2.small
2023-06-15 12:21:02 [ℹ]  eksctl version 0.144.0
2023-06-15 12:21:02 [ℹ]  using region ap-southeast-2
2023-06-15 12:21:03 [ℹ]  setting availability zones to [ap-southeast-2c ap-southeast-2a ap-southeast-2b]
2023-06-15 12:21:03 [ℹ]  subnets for ap-southeast-2c - public:192.168.0.0/19 private:192.168.96.0/19
2023-06-15 12:21:03 [ℹ]  subnets for ap-southeast-2a - public:192.168.32.0/19 private:192.168.128.0/19
2023-06-15 12:21:03 [ℹ]  subnets for ap-southeast-2b - public:192.168.64.0/19 private:192.168.160.0/19
2023-06-15 12:21:03 [ℹ]  nodegroup "ng-e2cc3119" will use "" [AmazonLinux2/1.25]
2023-06-15 12:21:03 [ℹ]  using Kubernetes version 1.25
2023-06-15 12:21:03 [ℹ]  creating EKS cluster "kube-cluster" in "ap-southeast-2" region with managed nodes
2023-06-15 12:21:03 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2023-06-15 12:21:03 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-2 --cluster=kube-cluster'
2023-06-15 12:21:03 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "kube-cluster" in "ap-southeast-2"
2023-06-15 12:21:03 [ℹ]  CloudWatch logging will not be enabled for cluster "kube-cluster" in "ap-southeast-2"
2023-06-15 12:21:03 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-southeast-2 --cluster=kube-cluster'
2023-06-15 12:21:03 [ℹ]
2 sequential tasks: { create cluster control plane "kube-cluster",
    2 sequential sub-tasks: {
        wait for control plane to become ready,
        create managed nodegroup "ng-e2cc3119",
    }
}
2023-06-15 12:21:03 [ℹ]  building cluster stack "eksctl-kube-cluster-cluster"
2023-06-15 12:21:03 [ℹ]  deploying stack "eksctl-kube-cluster-cluster"
2023-06-15 12:21:33 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:22:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:23:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:24:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:25:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:26:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:27:03 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:28:04 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:29:04 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:30:04 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-cluster"
2023-06-15 12:32:05 [ℹ]  building managed nodegroup stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:32:05 [ℹ]  deploying stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:32:05 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:32:35 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:33:16 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:33:58 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:35:13 [ℹ]  waiting for CloudFormation stack "eksctl-kube-cluster-nodegroup-ng-e2cc3119"
2023-06-15 12:35:13 [ℹ]  waiting for the control plane to become ready
2023-06-15 12:35:14 [✔]  saved kubeconfig as "/home/ansadmin/.kube/config"
2023-06-15 12:35:14 [ℹ]  no tasks
2023-06-15 12:35:14 [✔]  all EKS cluster resources for "kube-cluster" have been created
2023-06-15 12:35:14 [ℹ]  nodegroup "ng-e2cc3119" has 2 node(s)
2023-06-15 12:35:14 [ℹ]  node "ip-192-168-52-233.ap-southeast-2.compute.internal" is ready
2023-06-15 12:35:14 [ℹ]  node "ip-192-168-85-0.ap-southeast-2.compute.internal" is ready
2023-06-15 12:35:14 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-e2cc3119"
2023-06-15 12:35:14 [ℹ]  nodegroup "ng-e2cc3119" has 2 node(s)
2023-06-15 12:35:14 [ℹ]  node "ip-192-168-52-233.ap-southeast-2.compute.internal" is ready
2023-06-15 12:35:14 [ℹ]  node "ip-192-168-85-0.ap-southeast-2.compute.internal" is ready
2023-06-15 12:35:16 [ℹ]  kubectl command should work with "/home/ansadmin/.kube/config", try 'kubectl get nodes'
2023-06-15 12:35:16 [✔]  EKS cluster "kube-cluster" in "ap-southeast-2" region is ready
```
Get information of Kubernetes Nodes
```
[ansadmin@EKS-Bootstrap-server ~]$ kubectl get node -o wide
NAME                                                STATUS   ROLES    AGE   VERSION               INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-52-233.ap-southeast-2.compute.internal   Ready    <none>   51m   v1.25.9-eks-0a21954   192.168.52.233   13.239.14.60     Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
ip-192-168-85-0.ap-southeast-2.compute.internal     Ready    <none>   51m   v1.25.9-eks-0a21954   192.168.85.0     54.252.193.170   Amazon Linux 2   5.10.179-168.710.amzn2.x86_64   containerd://1.6.19
```
 **And then we will prepare ansible playbook to create docker image, tag and push to docker hub.
 ```
 ansadmin@ansible-server:/opt/docker$ cat regapp.yml
---
- hosts: ansible

  tasks:
  - name: create docker image
    command: docker build -t regapp:latest .
    args:
      chdir: /opt/docker
  - name: create tag to push image to docker hub
    command: docker tag regapp:latest kyinaing/regapp:latest

  - name: push docker image to docker hub
    command: docker push kyinaing/regapp:latest
  ```
    
  And we can verify our playbook as below
    
  ```
    ansadmin@ansible-server:/opt/docker$ ansible-playbook regapp.yml --check

    PLAY [ansible] *************************************************************************************************************

    TASK [Gathering Facts] *****************************************************************************************************
    ok: [172.31.24.126]

    TASK [create docker image] *************************************************************************************************
    skipping: [172.31.24.126]

    TASK [create tag to push image to docker hub] ******************************************************************************
    skipping: [172.31.24.126]

    TASK [push docker image to docker hub] *************************************************************************************
    skipping: [172.31.24.126]

    PLAY RECAP *****************************************************************************************************************
    172.31.24.126              : ok=1    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```
Now we will prepare playbook to deploy container in eks cluster and verify.
```
ansadmin@ansible-server:/opt/docker$ vi kube-deploy.yml
---
- hosts: kubernetes
  #  become: true
  #user: root

  tasks:
  - name: deploy regapp on kubernetes
    command: kubectl apply -f regapp-deploy.yml
  - name: deploy regapp service on kubernetes
    command: kubectl apply -f regapp-service.yml
  ```
  Verify
  ```
ansadmin@ansible-server:/opt/docker$ ansible-playbook kube-deploy.yml --check

PLAY [kubernetes] ************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************
[WARNING]: Platform linux on host 172.31.7.117 is using the discovered Python interpreter at /usr/bin/python3.9,
but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.14/reference_appendices/interpreter_discovery.html for more information.
ok: [172.31.7.117]

```
# Create Job in Jenkins to push docker image to docker hub and to run container in eks cluster by eks bootstrap server

Creat new job in jenkins
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/22ac6473-f86c-42d9-a807-fa4a4ed9ae36)

Source Code Configuration
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/f9d430f1-78a6-4199-bffe-e904c117e8d2)

we can set the branch name of git
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/430609ad-61c4-4a5e-9dce-2789876e3cab)

For continous integration, we can set build triggers with Poll SCM.
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/71d6240a-f54f-47dd-b936-1426bf4fce48)
```
* * * * *
``` 

**This is meaning of "every minute" when you say "* * * * *". Perhaps we can also set "H * * * *" to poll once per hour**.

And check Ignore post-commit hooks
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/c67c84b4-f072-4e9a-9ab6-3abd4d372267)

Build Goals and Options to clean install
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/59fad53c-30e2-42e3-8e97-4c34d77fb256)

Finnally, we will configure post-build actions.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/a75a50e9-5bcd-42c9-bfe4-ce0275434da8)

For the initial run, we need to manually execute the build now command and it does not require any subsequent commit changes from Git because we configure build trigger.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/9ee6c6cf-daab-40b0-8471-67300d67e28b)










