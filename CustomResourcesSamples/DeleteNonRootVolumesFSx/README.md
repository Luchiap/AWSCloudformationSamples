# Delete non root volumes from AWS::FSx::StorageVirtualMachine

When a s3 bucket is deleted from cloudformation stack operations, the operation will fail if the s3 bucket has objects that were not deleted.

The same applies to FSx storage virtual machines. When a storage virtual machine is deleted from cloudformation stack operations, the operation will fail if the storage virtual machine has non root volumes that were not deleted.

## Solution





