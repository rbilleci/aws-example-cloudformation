

## Instructions

#### Get the AMI

Get the image id of the latest storage gateway image from the marketplace. Rune the following command, making sure to enter 
the region you intend to deploy to.

<code>
aws ec2 describe-images --region us-east-1 --owners amazon --filters 'Name=name, Values=aws-storage-gateway*'  --query 'Images[*].{date:CreationDate, image:ImageId} | sort_by(@, &date)[-1] | image'
</code>

#### Launch the CFN Template

First, make sure you have a keypair for the region, you will need to enter it into the cfn template.

From the AWS Management Console run the cfn.yaml template, making sure to update the AMI id's and other parameters. 

Get a coffee. It will take up to 30 minutes to create the template.

This sets up two VPCs:

- VPC A: Fleet of Squid proxy servers that connect to the VPC endpoints for the AWS Storage Gateway service and S3
- VPC B: Emulated on-premise environment, using VPC Peering to connect to the proxy servers in VPC A

#### Setup the AD Domain to forward DNS requests to the VPC

For the AWS Storage Gateway to find the VPC endpoints, you'll need to configure the Active Directory DNS servers to forward
requests to the VPC.

First, get the directory id of your directory server. You'll need to enter it into the commands below:

Then: run this command to forward all requests to ec2.internal:

<code>
aws ds create-conditional-forwarder --region us-east-1 --directory-id d-996726d0d0 --remote-domain-name ec2.internal --dns-ip-addrs 10.1.0.2 
</code>

Next, run this command to forward all requests to amazonaws.com:

<code>
aws ds create-conditional-forwarder --region us-east-1 --directory-id d-996726d0d0 --remote-domain-name amazonaws.com --dns-ip-addrs 10.1.0.2
</code>


#### Initialize the AWS Storage Gateway

In the AWS Management Console:

1. Get the public IP address of your AWS Storage Gateway instance
2. Get the endpoint address of the VPC Endpoint for the AWS Storage Gateway service
3. From the AWS Management Console, go the AWS Storage Gateway user interface
4. Create a new AWS Storage Gateway
    - Select "File Gateway"
    - For the host platform select "Amazon EC2"
    - For the endpoint type, select "VPC"
    - Provide the VPC endpoint address of the AWS Storage Gateway service
    - Provide the public IP address of the AWS Storage Gateway
    - Click "Connect to Gateway"
    - You can now provide a gateway name
    - Click "Activate Gateway"
    - Wait, then click on "Configure Logging"
    - Select the defaults and choose "Save and Continue"

#### Configure the AWS Storage Gateway to connect to the Active Directory Domain and Squid Servers

1. In the AWS Management Console: Get the DNS IP addresses of the Active Directory cluster (IPs, not dns name)
2. In the AWS Management Console: Get the endpoint address of your private NLB (dns name, not IP)
3. From the terminal: Login to the Storage Gateway. You can get the connection details from the EC2 user interface
    - Configure the Storage Gateway to use the DNS addresses taken from step 1.
    - Configure the Storage Gateway to use the Proxy Server. Use the address of the NLB taken from step 2. Make certain to use port 3128, the standard squid port
4. When prompted, make sure to restart the networking and router. Then you have to wait a few minutes
5. Use the Storage Gateway tool to test the network configuration and make sure all the tests pass

#### Join the AWS Storage Gateway to the AD Domain

In the AWS Management Console:

1. Go to the AWS Storage Gateway user interface
2. Select the Gateway from the list
3. Select "Actions", then "Edit SMB Settings"
4. Edit the "Active Directory settings" and enter your domain information
    - if you used defaults, you should only need to enter the domain, user, and password.
    - domain is: aws.com
    - user is: admin
    - password is: !SGExample1234!
5. make sure there are no error messages

#### Configure the file share

In the AWS Management Console:

1. Go to the AWS Storage Gateway user interface
2. Select the Gateway from the list
3. Select "Create File Share"
4. For the bucket name, make certain to enter the bucket that was created as part of the cloudformation stack
5. Enter a good fileshare name, like "file-share"
6. Select "Server Message Block SMB"
7. Select "Next, Next, Next, and Create File Share"

#### Test it out

A windows client was created with the cloudformation template. You can connect to it and test out the share, using the "aws.com/admin" user

e.g. net use E: \\10.1.2.14\file-share /U:aws.com\admin