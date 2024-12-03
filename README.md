# CrowdStrike Falcon Elastic Beanstalk Deployment

 This repository contains a sample AWS Elasticbeanstalk application to help illustrate how to deploy the Falcon sensor on your Elastic Beanstalk compute resources.

## Deployment Methods

 There are 3 main ways to deploy the Falcon sensor:

### AWS SSM

This method involves using AWS SSM distributor to deploy the Falcon sensor directly on the machine. An in depth guide can be found [in this repository](https://github.com/CrowdStrike/aws-ssm-distributor?tab=readme-ov-file)

### Baked in AMI

This method involves creating custom AMIs with the Falcon sensor already installed using [EC2 Image Builder](https://aws.amazon.com/image-builder/), and using that AMI for your Elasticbeanstalk compute resources. An in depth guide can be found here *to-do: add documentation reference*

### ebextensions

The recommended way to install the Falcon sensor for Elastic beanstalk workloads is by utilizing ebextensions.

#### Configuration

First, you will need to add the Falcon client id and secret to Secrets Manager.

Create a `script` folder. In the script folder, create the following file.

`install_falcon.sh`

```bash
#!/bin/bash

export FALCON_CLIENT_ID="$(aws secretsmanager get-secret-value --secret-id "$SECRET_ID" --query SecretString --output text | jq -r .FALCON_CLIENT_ID)"
export FALCON_CLIENT_SECRET="$(aws secretsmanager get-secret-value --secret-id "$SECRET_ID" --query SecretString --output text | jq -r .FALCON_CLIENT_SECRET)"

curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.7.1/bash/install/falcon-linux-install.sh | bash

```

This script retrieves the Falcon credentials from secrets manager, and installs the sensor using the linux script from the [falcon-scripts repository](https://github.com/CrowdStrike/falcon-scripts).

Create a `.ebextensions` directory in the root of the application package. Within this folder, create the following files.

`ssm.config`

```
option_settings:
  aws:elasticbeanstalk:application:environment:
    SECRET_ID: <secret name>
```

>[Note]: Remember to change \<secret name\> to your actual secret name

This config will ensure the secret name is exported as an environment variable.

`falcon.config`

```
container_commands:
  01_falcon:
    command: "sh scripts/install_falcon.sh"
```

This config will execute the falcon install script we previously created at runtime.

#### Deploying example app

##### Prerequisites

- Falcon Client ID
- Falcon Client Secret
- [aws-elasticbeanstalk-service-role](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts-roles-service.html)
- Instance Profile

##### Store Falcon credentials

- Open the AWS console ans navigate to Secrets Manager.
- Click on **Store a new secret**.
- Under **Secret type** Choose **Other type of secret**
- Create an entry for **FALCON_CLIENT_ID** and **FALCON_CLIENT_SECRET** with their respective values
- Under **Secret Name** enter **ebs/falcon/credentials**
- Click **Next** -> **Next** -> **Store**

##### Create a key-pair

>[Note]: If you already have a key pair you want to use, feel free to skip this step

In the AWS Console, navigate to **EC2**

Under **Network and security** click **Key Pairs**

Click **Create key pair**

Give your key pair a suitable name.

Click **Create key pair** as save your key to a secure location.

##### Create an Instance profile

Navigate to **IAM** in the AWS console.

Click **Roles** and **Create role**

Under **Select trusted entity** choose **AWS Service**

Under **Use case** choose **EC2** then click **Next**

Select the **AWSElasticBeanstalkWebTier** and **SecretsManagerReadWrite** managed policies. Click **Next**

Give your role a name then click **Create role**

##### Create Elastic Beanstalk Environment

Navigate to **Elastic Beanstalk** in the AWS Console
Click on **Create environment**

- **Configure Environment**
  - Choose the following values
    - **Environment tier**: Web server environment
    - **Application Name**: CSHelloWorld
    - **Platform**: Python
    - **Platform branch**: Python 3.12
    - **Platform version**: 4.3.1(Recommended)
  - Click **Next**
- **Configure service access**
  - For **Existing service roles** choose **aws-elasticbeanstalk-service-role**
    - If you do not have this role, refer to [this documentation](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts-roles-service.html) to get it deployed.
  - For **EC2 key pair**, choose the key pair we created earlier
  - For **EC2 instance profile** choose the instance profile we created earlier
  - Click **Next** until you arrive at **Configure instance traffic and scaling**
- **Configure instance traffic and scaling**
  - For **Root volume type** choose **General Purpose 3(SSD)**
  - Leave everything else as default, scroll down, and click **Skip to review**
  - Click **Submit**

##### Package the sample application

Open up this root directory in your terminal and execute the following commands

```bash
zip -r ./hello_world.zip .
```

##### Deploy the sample application

Navigate back to the environment we created in Elastic Beanstalk.

Click **Upload and deploy**.

Select the **hello_world.zip** file we created.

Click **Deploy**

##### Verify the sample application was deployed correctly

Click the url under **Domain**

You should see a simple webpage with a `Hello World!` message

#### Verify the Falcon sensor was installed correctly

In the logs tab, click **Request logs** and then **Full**

Download and unzip the contents

In the `var/log/` directory, open the cfn-init-cmd.log file. At the bottom of the file, you should see some logs that resemble the following

```
...
2024-12-03 17:53:01,286 P3644 [INFO] Command 01_falcon
2024-12-03 17:53:14,245 P3644 [INFO] -----------------------Command Output-----------------------
2024-12-03 17:53:14,246 P3644 [INFO]    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
2024-12-03 17:53:14,247 P3644 [INFO]                                   Dload  Upload   Total   Spent    Left  Speed
2024-12-03 17:53:14,247 P3644 [INFO]  
2024-12-03 17:53:14,247 P3644 [INFO]    0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
2024-12-03 17:53:14,247 P3644 [INFO]  100 35817  100 35817    0     0   164k      0 --:--:-- --:--:-- --:--:--  164k
2024-12-03 17:53:14,247 P3644 [INFO]  Check if Falcon Sensor is running ... [ Not present ]
2024-12-03 17:53:14,247 P3644 [INFO]  Falcon Sensor Install  ... 
2024-12-03 17:53:14,247 P3644 [INFO]  Installed:
2024-12-03 17:53:14,247 P3644 [INFO]    falcon-sensor-7.20.0-17306.amzn2023.x86_64                                    
2024-12-03 17:53:14,247 P3644 [INFO]  
2024-12-03 17:53:14,247 P3644 [INFO]  [ Ok ]
2024-12-03 17:53:14,247 P3644 [INFO]  Falcon Sensor Register ... [ Ok ]
2024-12-03 17:53:14,247 P3644 [INFO]  Falcon Sensor Restart  ... [ Ok ]
2024-12-03 17:53:14,247 P3644 [INFO]  Falcon Sensor installed successfully.
2024-12-03 17:53:14,247 P3644 [INFO] ------------------------------------------------------------
2024-12-03 17:53:14,248 P3644 [INFO] Completed successfully.

```

Congratulations! The sensor was successfully installed.

You can further confirm this by querying for your instance in the Falcon console.
