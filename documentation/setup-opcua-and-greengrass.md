# Setup OPC-UA Server & AWS IoT Greengrass

For this lab the OPC-UA and AWS IoT Greengrass will be installed & pre-configured for you - as it may take quite a while to set them up for the first time. However, if you'd like to install the server yourself, follow these steps:

## Setup OPC-UA Server

1. Create a VPC with a public subnet (or use the default VPC)

2. Create a security group and allow access from your current IP address on port 3389. Call the security group `opcua-sg`.

3. Create an EC2 instance with the following configuration:

- t2.medium / t3.medium instance type
- Windows Server 2019 operation system

Place the instance in the VPC & attach the `opcua-sg` security group to it.

Remember to create & download the access key that you'll use to decrypt Administrator's password for the instance!

3. Log into machine using RDP Client.

4. Download the KEPServerEX demo & install it.

[Download page](https://www.ptc.com/en/products/kepware/kepserverex/demo-download)

5. Launch the KEPServerEX.

6. Configure OPC-UA simulator. You may use information from this post: [Setting Up OPC-UA server](https://aws.amazon.com/blogs/iot/collecting-organizing-monitoring-and-analyzing-industrial-data-at-scale-using-aws-iot-sitewise-part-1/)

## Setup AWS IoT Greengrass

1. Create a security group called `greengrass-sg` in the previously set up VPC.

2. Add 2 rules to the previously created `opcua-sg` security group that allow inbound connections over TCP & UPD on all ports from the `greengrass-sg` security group.

3. Create an EC2 instance with the following configuration:

- t2.medium / t3.medium instance type
- Ubuntu 20.04 version
- IAM Role with the managed policy `AmazonSSMManagedInstanceCore` (we will connect to the instance using AWS Systems Manager - Session Manager) and 2 Customer managed policies:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iotsitewise:BatchPutAssetPropertyValue",
            "Resource": "*"
        }
    ]
}
```

and:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateTokenExchangeRole",
            "Effect": "Allow",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:CreatePolicy",
                "iam:CreateRole",
                "iam:GetPolicy",
                "iam:GetRole",
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::YOUR-AWS-ACCOUNT-ID:role/GreengrassV2TokenExchangeRole",
                "arn:aws:iam::YOUR-AWS-ACCOUNT-ID:policy/GreengrassV2TokenExchangeRoleAccess"
            ]
        },
        {
            "Sid": "CreateIoTResources",
            "Effect": "Allow",
            "Action": [
                "iot:AddThingToThingGroup",
                "iot:AttachPolicy",
                "iot:AttachThingPrincipal",
                "iot:CreateKeysAndCertificate",
                "iot:CreatePolicy",
                "iot:CreateRoleAlias",
                "iot:CreateThing",
                "iot:CreateThingGroup",
                "iot:DescribeEndpoint",
                "iot:DescribeRoleAlias",
                "iot:DescribeThingGroup",
                "iot:GetPolicy"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DeployDevTools",
            "Effect": "Allow",
            "Action": [
                "greengrass:CreateDeployment",
                "iot:CancelJob",
                "iot:CreateJob",
                "iot:DeleteThingShadow",
                "iot:DescribeJob",
                "iot:DescribeThing",
                "iot:DescribeThingGroup",
                "iot:GetThingShadow",
                "iot:UpdateJob",
                "iot:UpdateThingShadow"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::YOUR-IIOT-BUCKET_NAME",
                "arn:aws:s3:::YOUR-IIOT-BUCKET_NAME/*"
            ]
        }
    ]
}
```

4. Install and configure AWS IoT Greengrass Core software following these instructions:

[Documentation](https://docs.aws.amazon.com/greengrass/v2/developerguide/getting-started.html)

Installing command example:

``` bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --aws-region eu-west-1 \
  --thing-name iot-in-clouds-2022-core-device \
  --thing-group-name iot-in-clouds-2022-group \
  --thing-policy-name iot-in-clouds-2022-core-device-policy \
  --tes-role-name iot-in-clouds-2022-core-device-token-exchange-role \
  --tes-role-alias-name iot-in-clouds-2022-core-device-token-exchange-role-alias \
  --component-default-user ggc_user:ggc_group \
  --provision true \
  --setup-system-service true
```

5. In the AWS console install the necessary components:

a. Go to AWS IoT Core
b. From the menu on the left select Greengrass devices - Deployments
c. Create a new deployment and select the core device as a target type
d. Select these components to install:

```text
aws.greengrass.Cli
aws.greengrass.StreamManager
aws.iot.SiteWiseEdgeCollectorOpcua
aws.iot.SiteWiseEdgePublisher
```

e. Select `Next` (several times) & deploy components.

6. Log into the instance and make sure that all is up and running.

7. Check connectivity between the greengrass instance and the opcua instance

**Remember to open traffic on the Windows firewall from and to your greengrass device!**

**Remember to set the client certificate as trusted**

**Remeber to switch off authentication by username & password**

8. Setup the Edge gateway & data source in AWS IoT Sitewise


9. [Optional] Enable ingestion of data streams not associated with asset properties

```bash
aws iotsitewise put-storage-configuration \
    --storage-type MULTI_LAYER_STORAGE \
    --disassociated-data-storage ENABLED
```
