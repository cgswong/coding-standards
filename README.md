# Coding Standards

## Lambda Functions

- Initialize SDK clients / DB connections outside the handler, taking advantage of execution context.

```python
import boto3
import logging

client = boto3.client('ec2')
LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)

def lambda_handler(event, context):
  # code here
```

- Avoid potential data leaks: do not use the execution context to store data, events, or other information with security implications
- Follow the principle of least privilege for the Lambda execution role. Sharing an IAM role within more than one Lambda function will violate least-privileged access.
- Setting reserved/provisioned concurrency is recommended for critical functions so they are not throttled.
- Use a standard header as part of an initial comment. It should summarize what the function does, who wrote it, etc.
- Import what you require, i.e. be precise. For example, instead of importing entire botocore for ClientError, try the following:

```python
from botocore.exceptions import ClientError
```

- Use log levels judiciously and avoid “print” to help fine-tune the function log output. Example:

```python
import logging

LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)

LOGGER.info("starting code execution")
LOGGER.error("function failed")
```

- Modularity is recommended, so divide your script logically by functions to make it more reusable. Separate the handler from your core logic. Example:

```python
def lambda_handler(event, context):

# call other functions

def get_value(event):
  # get value

def operation1(param):
  # code for operation1

def operation2(param):
  # code for operation2
```

- Follow standards for writing the code in any language. For an example for Python, follow PEP 8.
- Handle exceptions in all situations! Example:

```python
try:
  # code goes here
except ClientError as e:
  err = e.response['Error']
  LOGGER.error("Error {} occurred".format(err))
  ```
  
## CloudFormation

### General

- Divide/layer infrastructure logically: instead of creating a single large stack, break it down into smaller components and run them as nested stacks.

Example: 

```shell
MyInfra
|__webserver.yaml
|__appserver.yaml
|__database.yaml
```

- Validate templates before using them to create or update stacks: this helps to catch errors before any resources are created.
- Never modify resources created via stacks directly from the console: always use CloudFormation itself by updating said stack to keep resources and code in sync.
- Create change sets before a stack is updated: this helps in predicting the impact of updating the stack. 
- Restrict actions users can perform via IAM roles. A point to note here is that there is a separate service role for CloudFormation that the service uses to make API calls to create resources instead of the user’s assumed role.
- Set up a stack policy for critical mission resources such that a user may not make changes to the stack: the stack can further be protected by adding conditions. Details are provided [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-template.html#using-iam-template-conditions).

Example:

```json
{
  "Statement":[
    {
      "Effect":"Allow",
      "Action":"Update:*",
      "Principal":"*",
      "Resource":"*"
    },
    {
      "Effect":"Deny",
      "Action":"Update:*",
      "Principal":"*",
      "Resource":"LogicalResourceId/ProductionServer"
    }
  ]
}
```

### Writing CloudFormation Templates

- Apply reusable stacks, which are the best type of stacks. Parameters, conditions, and mappings help to create them.

```yaml
Parameters: 
  EnvironmentType: 
    Description: The environment type
    Type: String
    Default: dev
    AllowedValues: 
      - dev
      - uat
    ConstraintDescription: must be a dev or uat
Mappings: 
  RegionToInstance: 
    ap-south-1: 
    dev: "t2.micro"
    uat: "m5.large"
Conditions:
  CreateProdInstance: !Equals [ !Ref EnvType, prod ]
Resources:
  EC2Instance:
  Type: "AWS::EC2::Instance"
  ***
```

- Utilize “export” and “import” to share values between stacks. A tricky exercise caution is to use them for parameters that are not prone to change like VPC IDs and Subnet IDs. Do not export and import the parameters in the same stack. This can cause problems when updating and deleting the stacks.

```yaml
# Export
Outputs:
  Logical ID:
    Description: Information about the value
    Value: Value to return
    Export:
      Name: Value to export
      
# Import
Fn::ImportValue: *sharedValueToImport*
```

- Do not hardcode credentials or sensitive data when writing templates. Use dynamic references such as SSM Parameter Store for configuration data or Secrets Manager for sensitive data.

Example:

```yaml
#ssm
MyS3Bucket:
Type: AWS::S3::Bucket
Properties:
   AccessControl: '{{resolve:ssm:S3AccessControl:1}}'
      
#ssm-secure[not available for all resources, check before using]
MyIAMUser:
Type: AWS::IAM::User
Properties:
    UserName: UserName
    LoginProfile:
    Password: '{{resolve:ssm-secure:IAMUserPassword:10}}'
```

- Look [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html) for a full list of helpful parameter types.

Example:

```yaml
Parameters: 
  myKeyPair: 
    Description: Amazon EC2 Key Pair
    Type: "AWS::EC2::KeyPair::KeyName"
  mySubnetIDs: 
    Description: Subnet IDs
    Type: "List<AWS::EC2::Subnet::Id>"
```

- Apply parameter constraints where possible to weed out invalid data while the template is being executed.

Example:

```yaml
Name: 
  Description: 'Enter the name of your Instance'
  Type: String
  MinLength: 1
  MaxLength: 5
    AllowedPattern: ^[a-zA-Z0-9]*$
```

- Utilize the metadata section to include JSON or YAML object, which provides details about the template.

```yaml
Metadata:
  Instances:
    Description: "Information about the instances"
```

- Use interface property (_AWS::CloudFormation::Interface_) to sort/group the parameters logically. 

Example:

```yaml
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Network Configuration
        Parameters:
          - VPCID
          - SubnetId
          - SecurityGroupID
      - Label:
          default: Amazon EC2 Configuration
        Parameters:
          - InstanceType
          - KeyName
    ParameterLabels:
      VPCID:
        default: Which VPC should this be deployed to?
```

- Provide comments when using YAML. This makes the template easily readable.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: This is a template
Resources:
  MyEC2Instance: #An inline comment
    Type: "AWS::EC2::Instance"
    Properties: 
      ImageId: "ami-0ff8a91507f77f867" #Another comment -- This is a Linux AMI
```

- Use `NoEcho` property to obfuscate any sensitive parameter values like password. Note, this will still be available when viewed via CloudTrail.

```yaml
Pwd: 
  NoEcho: true
  Description: The account password
  Type: String
  MinLength: 1
  MaxLength: 12
  AllowedPattern: ^[a-zA-Z0-9]*$
```
