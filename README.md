************CIT481_FinalProject: Setting up a Kubernetes Cluster on Spot Instances****************

************************************CREATE AN AWS ACCOUNT****************************************
1. Create an AWS account with Administrator access here: https://aws.amazon.com/getting-started/
2. Create an IAM user with Administrator access to the AWS account.
3. Fill in the information for "User name*", check the box for "AWS Management Console access" under "Access Type", enter a "Console Password*", UNcheck the box "Require password reset", and select "Next:Permissions".
4. Select "Attach existing policies directly", check the "AdministratorAccess" policy, and select "Next:Review". 
5. Select "Create user" and REMEMBER to SAVE the login url that's listed on the next page. You can also choose to "Download .csv" which downloads a credentials.csv file that lists the login url. 





************************************CREATE A WORKSPACE*******************************************
1. Create a Cloud9 environment in your closest region by clicking this link: https://us-west-2.console.aws.amazon.com/cloud9/home?region=us-west-2
2. Select "Create environment", name your workspace as desired, create a VPC if you don't already have one available. For test purposes, I created a VPC with the option of a single subnet. Select the VPC you'd like to use, select the subnets of your liking, and select "Create environment" to initialize the build. 





******************************CREATE AN IAM ROLE FOR YOUR WORKSPACE*******************************
1. To create an IAM Role with Administrator access go to this link: https://us-west-2.console.aws.amazon.com/cloud9/home?region=us-west-2
2. Make sure "AWS service" and "EC2" are both selected. Select "Next:Permissions". 
3. Confirm "AdministratorAccess" is selected and click "Next".
4. Enter a name for the role and select "Create role". 





***********************************ATTACH IAM ROLE TO WORKSPACE************************************
1. Navigate to your Cloud9 EC2 Instance.
2. Select the instance and go to Actions dropdown menu, choose Instance Settings, and click Attach/Replace IAM Role.
3. Choose the name of the IAM Role you created with Administrator access from the "IAM role*" dropdown menu and select "Apply".





******************************************CREATE AN SSH KEY****************************************
1. Open up your Cloud9 IDE and create a New terminal. 
2. Run the command below to generate a SSH Key which allows secure shell access to the worker nodes if necessary:

$ ssh-keygen

3. Press "Enter" three times to select defaults.
4. Upload the public key to your EC2 region by running this command:

$ aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub





************************************DOWNLOAD EKSCTL BINARY**************************************
1. Download the eksctl binary by entering the entire command below into your Cloud9 workspace:

$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin

2. Confirm the eksctl command works by entering:

$ eksctl version





***********************************VALIDATE THE IAM ROLE*************************************
1. Use the command below to validate that Cloud9 IDE is using the correct IAM role:

$ aws sts get-caller-identity

2. The Assumed role name (Arn) for your results should be the name of the IAM role you created or TeamRole, if you see anything other than that, it's invalid and must be changed in order to move on. 
3. To make the proper changes, navigate back to your workspace and select the gear button to open up the Preferences tab. 
4. In the menu on the left, select AWS Settings and turn OFF "AWS managed temporary credentials".
5. Close the Preferences tab.
6. Remove any existing credential files by typing in this command:

$ rm -vf ${HOME}/.aws/credentials

7. Configure AWS CLI with current region as default by entering :

$ export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region

8. Confirm the IAM role by re-entering the command above:

$ aws sts get-caller-identity





***********************************INSTALL KUBERNETES TOOLS**********************************
1. Create ~/.kube to store kubectl configurations:

$ mkdir -p ~/.kube

2. Install kubectl:

$ sudo curl --silent --location -o /usr/local/bin/kubectl "https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl"

sudo chmod +x /usr/local/bin/kubectl

3. Install AWS IAM Authenticator:

$ go get -u -v github.com/kubernetes-sigs/aws-iam-authenticator/cmd/aws-iam-authenticator
sudo mv ~/go/bin/aws-iam-authenticator /usr/local/bin/aws-iam-authenticator

4. Install JQ and envsubst:

$ sudo yum -y install jq gettext

5. Verify that the binaries are in their path and executing:

$ for command in kubectl aws-iam-authenticator jq envsubst
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done





***********************************CLONE SERVICE REPOS***************************************
1. Use the command below:

$ cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git





************************************CREATE EKS CLUSTER**************************************
1. Create an EKS Cluster by using this command:

$ eksctl create cluster --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=${AWS_REGION}

2. Test the cluster by confirming the nodes:

$ kubectl get nodes

3. Export the Worker Role Name to use throughout the workshop:

$ INSTANCE_PROFILE_NAME=$(aws iam list-instance-profiles | jq -r '.InstanceProfiles[].InstanceProfileName' | grep nodegroup)
INSTANCE_PROFILE_ARN=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Arn')
ROLE_NAME=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Roles[] | .RoleName')
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.bash_profile
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.bash_profile





***********************************DEPLOY KUBERNETES DASHBOARD*********************************
1. Deploy Kubernetes dashboard using following command:

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

2. Proxy requests to dashboard service:

$ kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &

3. To access the dashboard, within your Cloud9 environment click Tools > Preview > Preview Running Application.
4. At the end of the url attach the line below and hit enter to navigate to the website:

/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

5. Open a new terminal and enter:

$ aws-iam-authenticator token -i eksworkshop-eksctl --token-only

6. Copy the output of the command, click the radio button to select "Token", paste the output into the "Enter token" field, and click SIGN IN. 
