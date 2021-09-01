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

## Python

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
