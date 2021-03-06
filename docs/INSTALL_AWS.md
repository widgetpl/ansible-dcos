# Steps for DC/OS installation with Terraform and Ansible on AWS

With the following guide, you are able to install a DC/OS cluster on AWS. You need the tools Terraform and Ansible installed. On MacOS, you can use [brew](https://brew.sh/) for that.

```
brew install terraform
brew install ansible
```

## Setup infrastructure

### Pull down the DC/OS Terraform scripts below

```bash
make aws
```

### Configure your AWS ssh Keys

In the file `desired_cluster_profile` there is a `key_name` variable. This key must be added to your host machine running your terraform script as it will be used to log into the machines to run setup scripts. The default is `default`. You can find aws documentation that talks about this [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws).

When you have your key available, you can use ssh-add.

```bash
ssh-add ~/.ssh/path_to_you_key.pem
```

### Configure your IAM AWS Keys

You will need your AWS aws_access_key_id and aws_secret_access_key. If you dont have one yet, you can get them from the AWS documentation [here](
http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html). When you finally get them, you can install it in your home directory. The default location is `$HOME/.aws/credentials` on Linux and OS X, or `"%USERPROFILE%\.aws\credentials"` for Windows users.

Here is an example of the output when you're done:

```bash
$ cat ~/.aws/credentials
[default]
aws_access_key_id = ACHEHS71DG712w7EXAMPLE
aws_secret_access_key = /R8SHF+SHFJaerSKE83awf4ASyrF83sa471DHSEXAMPLE
```

### Terraform deployment

The setup variables for Terraform are defined in the file `desired_cluster_profile`. You can make a change to the file and it will persist when you do other commands to your cluster in the future.

For example, you can see the default configuration of your cluster:

```bash
$ cat desired_cluster_profile
os = "centos_7.4"
state = "none"
#
num_of_masters = "1"
num_of_private_agents = "3"
num_of_public_agents = "1"
#
aws_region = "us-west-1"
aws_bootstrap_instance_type = "m4.large"
aws_master_instance_type = "m4.2xlarge"
aws_agent_instance_type = "m4.2xlarge"
aws_public_agent_instance_type = "m4.2xlarge"
ssh_key_name = "default"
# Inbound Master Access
admin_cidr = "0.0.0.0/0"
```

You can plan the profile with Terraform while referencing:

```bash
make plan
```

If you are happy with the changes, the you can apply the profile with Terraform while referencing:

```bash
make launch-infra
```

## Install DC/OS

Once the components are created, we can run the Ansible script to install DC/OS on the instances.

The setup variables for DC/OS are defined in the file `group_vars/all`. Copy the example file, by running:

```
cp group_vars/all.example group_vars/all
```

The now created file `group_vars/all` is for configuring DC/OS. The variables are explained within the file. In order to setup DC/OS for AWS, you should change at least the following variables:

Change the exhibitor backend to `aws_s3`. So the master discovery is done by using an S3 bucket:

```
# Configuration for the Exhibitor Storage Backend
# options: static, aws_s3, azure
exhibitor: aws_s3
```
You also have to create an S3 bucket on your own and specify the AWS credentials, the bucket name, and the bucket region:

```
# AWS S3 Credentials (only needed for exhibitor: aws_s3)
aws_access_key_id: "YOUR_AWS_ACCESS_KEY_ID"
aws_secret_access_key: "YOUR_AWS_SECRET_ACCESS_KEY"
aws_region: YOUR_BUCKET_REGION
s3_bucket: YOUR_BUCKET_NAME
```

Ansible also needs to know how to find the instances that got created via Terraform.  For that we you run a dynamic inventory script called `./inventory.py`. To use it specify the script with the parameter `-i`. In example, check that all instances are reachable via Ansible:

```
ansible all -i inventory.py -m ping
```

Finally, you can install DC/OS by running:

```
ansible-playbook -i inventory.py plays/install.yml
```

## Access the cluster

If the installation was successful. You should be able to reach the Master load balancer. You can find the URL of the Master LB with the following command:

```
make ui
```

The terraform script also created a load balancer for the public agents:

```
make public-lb
```

## Destroy the cluster

To delete the AWS stack run the command:

```
make destroy
```
