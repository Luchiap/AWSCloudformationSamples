# Delete non root volumes from AWS::FSx::StorageVirtualMachine

## Premise

When a s3 bucket is deleted from cloudformation stack operations, the operation will fail if the s3 bucket has objects that were not deleted.

The same applies to FSx storage virtual machines. When a storage virtual machine is deleted from cloudformation stack operations, the operation will fail if the storage virtual machine has non root volumes that were not deleted.

## Solution

To solve this, we will make use of custom resources. With custom resources we have access to almost all AWS APIs. This allows us to circumvent CloudFormation resource limitations.

For this particular case we will make use of Python and Boto3. 

Boto3 APIs in question are:
- describe_volumes https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/fsx/client/describe_volumes.html
- delete_volume https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/fsx/client/delete_volume.html







