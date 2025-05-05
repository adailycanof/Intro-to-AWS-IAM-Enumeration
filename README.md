## PWNED LABS
# Intro to AWS IAM Enumeration
https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration

# Secenario
You are a security consultant hired by the global logistics company Huge Logistics. Following suspicious activity, you’ve been tasked with enumerating the IAM user dev01 and identifying any potentially compromised resources. Your mission is to examine and evaluate IAM roles, policies, and permissions.

# Team
Blue

# Difficulty
Beginner

# Real World
IAM (Identity and Access Management) is central to building, defending, and attacking cloud services. Both offensive and defensive security practitioners need a solid understanding of IAM and how to enumerate permissions. Attackers look for overly permissive settings or misconfigurations in a potential attack chain, while defenders need to enforce the principle of least privilege and identify any resources or services within the blast radius of a compromised IAM user.

# Walkthrough

Login to the AWS console with the credentails provided
username:Redacted!
password:Redacted!
accountNo: Redacted!
Access key ID: Redacted!
Secret access key: Redacted!

![used serices](image.png)

First point of call would be IAM — let's see what this user has access to.

![denied permissions](image-1.png)

From the image above, you can see that this user has very little access... mmmmmm. My last check will be GuardDuty. Since IAM is global, there's not much point in looking deeper into users, roles, or policies, as it's all denied. GuardDuty lists one new finding —

![ssh finding](image-2.png)

Lets keep this noted down:
44.219.62.158 is performing SSH brute-force attacks against i-0f9368c3dc7714c42. Looks like an EC2 instance needs a bit of poking. Let's come back to this in a moment. After looking around, I would pivot and use the AWS CLI to see if there's something more detailed I can find. If it's not already installed, then get the AWS CLI.

<bash>apt install awscli<bash>

After this has been installed run the following:

<bash>aws configure<bash>

![alt text](image-3.png)

After this has been setup run 

<bash>aws sts get-caller-identity<bash>

![alt text](image-4.png)

The above command is very useful — it's basically the AWS equivalent of ipconfig /all or pwd, etc. It's helpful for showing you which role is currently being used. Another useful command would be:

<bash>aws iam get-user<bash>

![alt text](image-5.png)

This is intresting, Its showing that when this user was created a tag was also used. let see who/what else has this tag attached.

![alt text](image-6.png)

Lets drop a bit deeper in and see what attached polocies this user has attached 

<bash>aws iam list-attached-user-policies --user-name dev01<bash>

![alt text](image-7.png)

Im curious, Lets see if there are any other users that we can see. There is dobut to see what else is visable given the access dev01 has but lets see. 

![alt text](image-8.png)

And Nothing!. Lets carry on, We have permissions so we can enumarate them. This shows that our IAM user has the AmazonGuardDutyReadOnlyAccess managed policy and a custom policy called dev01 attached. GuardDuty helps detect threats by keeping an eye out for suspicious or unauthorized activity in our AWS environment. We almost already knew this from what we could see from the console. 

Now lets check for inline polocies. 

<bash>aws iam list-user-policies --user-name dev01<ash>

![alt text](image-9.png)

Oooooo S3!, so there is an inlice policy named S3 that this user has access to. Lets see if I can see anyting at the root level 

<bash>aws s3 ls<bash>

Nothing, Ok, Lets have a closer look at the guarddity policy from above. Before looking at the policy lets check out how many versions this policy has had.

<bash>aws iam list-policy-versions --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess<bash>

![alt text](image-10.png)

Amazon and customer-managed policies can have many versions, allowing for reviews and rollbacks to previous policies. As a note, inline policies do not support versioning. The assumption is that version v4 is currently in use. Let’s check.

<bash>aws iam get-policy-version --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess --version-id v4<bash>

![alt text](image-11.png)

The Describe permission lets us pull info about GuardDuty — like detectors, findings, or how things are set up. Anything in the API that starts with "Describe" is fair game. Get is for grabbing specific details, like findings or threat intel sets. And List just gives us a list of things like detectors or members, but without all the deep details.

Let’s list out all the versions.

<bash>aws iam list-policy-versions --policy-arn arn:aws:iam::794929857501:policy/dev01<bash>

![alt text](image-12.png)

Can you see? Now lets see what policy v7 on dev01 can do.

<bash>aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/dev01 --version-id v7<bash>

![alt text](image-13.png)

There’s a nice tool called IAMCTL that provides a simple, human-readable output of IAM roles and policies. To install it, run:

<bash>pip install git+ssh://git@github.com:aws-samples/aws-iamctl.git<bash>

After mapping out the allowed actions, there might be something here. This custom policy attached to the dev01 user lets them see details about themselves, any groups they’re in, and what policies they have. It also gives access to check out the BackendDev role, see which policies are attached to it, and look up info on a policy called BackendDevPolicy (which we’re assuming is linked to that role). Plus, it allows viewing details about an AWS-managed policy too. Let's check.

<bash>aws iam list-attached-role-policies --role-name BackendDev<bash>

Ah ha! Let's dig that bit deeper. The role named BackendDev looks like a sudoers command on Linux — it seems to grant temporary access. This would give us privileged context within the role.

![alt text](image-14.png)


And gotcahya 

![alt text](image-15.png)

lets assume the BackEndDevRole

<bash>aws sts assume-role \
  --role-arn arn:aws:iam::794929857501:role/BackendDev \
  --role-session-name backend-dev-session<bash>

<bash>aws iam get-user-policy --user-name dev01 --policy-name S3_Access<bash>

![alt text](image-16.png)

<bash>aws s3 ls s3://hl-dev-artifacts<bash>

And there it is, The flag.