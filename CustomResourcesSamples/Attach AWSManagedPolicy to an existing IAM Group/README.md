## Summary

Cloudformation does not natively support attaching an AWS Managed Policy to an existing IAM Group. This example shows how to achieve this with Lambda and custom resources.

## Main Text

When CloudFormation does not support a particular property in a resource even though it's available via the AWS API, we can use custom resources to workaround it while the feature request is implemented.

This also serves as another example on how to use custom resources as it can be confusing, and how to use Lambda functions with Python.

## Pre-requisites

    Create an IAM User Group in the console, or have one available already
    Have an arn ready for an AWS Managed Policy. For example, for 'AdministratorAccess' the arn is 'arn:aws:iam::aws:policy/AdministratorAccess'
    Copy - Paste the provided code [1] into a text file. The code was also attached into this article

The following Template performs these actions:

    Takes as a parameter an already existing IAM Group as mentioned in the pre-requisites
    Takes as a parameter an AWS managed Policy as mentioned in the pre-requisites
    Creates an IAM Role (AWS::IAM::Role) to give permissions to the Lambda function
    Creates a Lambda function (AWS::Lambda::Function) with inline code in python
    Creates environment variables from the parameters inputs. Then in the lambda code section, those variables are retrieved and used.
    Attaches the inputted AWS Managed Policy during stack creation and detaches it during stack deletion.
    Saves a status in the output section of the stack. This status will depend on current stack condition. Also retrievable in CloudWatch logs monitoring of the lambda function.


## Parameters Section

In this section, during stack creation you will provide two parameters. An already existing IAM User Group and an AWS Managed Policy of your choice.

#Parameter section, during stack creation I will input an existing IAM Group name
Parameters:
  GroupNameEnv:                             #Existing Group that you input
    Type: String
  ARNAWSManagedPolicyToAttach:              #Any AWS managed policy that you input as ARN, for example arn:aws:iam::aws:policy/AdministratorAccess
    Type: String


## Resources Section

In this first part, we define the lambda function with the following properties, we reference a role which we create in this same stack and we add as environment variables the parameters we defined before. This is needed to pass these values into the code section of the lambda. The runtime is python 3.9.

FunctionGroupAttachAndDetach:
    Type: 'AWS::Lambda::Function'
    Properties:
      Handler: index.handler
      Role: !GetAtt LambdaCustomResourceRoleGroupAttachAndDetach.Arn
      Environment:
        Variables: 
          GroupNameEnv : !Ref GroupNameEnv
          AWSARNManagedPolicyToAttachEnv: !Ref ARNAWSManagedPolicyToAttach
      Runtime: python3.9

In this second part, we define the inline code, see provided comments for further detailed information.
Boto3 was used [2].

#Python code
      Code:
        ZipFile: !Sub |
            import os                                                                                                                     #needed to 'import' the Environment variable
            import boto3                                                                                                                  #needed to use custom resources
            import cfnresponse                                                                                                            #needed to use custom resources        
            GroupNameVar = os.environ['GroupNameEnv']                                                                                     #Importing the Group name and saving it as GroupNameVar
            AWSARNManagedPolicyToAttachVar = os.environ['AWSARNManagedPolicyToAttachEnv']                                                 #Importing the Managed policy arn and saving it as ARNManagedPolicyToAttachVar
            client = boto3.client('iam')                                                                                                  #Importing iam custom resources
            def handler(event, context):
              try:
                if event['RequestType'] == 'Delete':                                                                                      #When in cloudformation we perform delete stack event
                  response = client.detach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                  #we detach the policy to the group 
                  ResponseBody = {'Status': 'Deletion Complete'}                                                                          #Creating a 'Status' state, so we can use it output (optional)
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, ResponseBody)
                  return
                elif event['RequestType'] == 'Create':                                                                                    #When in cloudformation we perform create stack event
                  response = client.attach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                  #we attach the policy to the group  
                  ResponseBody = {'Status': 'Creation Complete'}                                                                          #Creating a 'Status' state, so we can use it output (optional), could also use ResponseBody = Response  
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, ResponseBody)      
              except Exception:                                                                                                           #If syntax is wrong, or it fails for any reason, an exception will take us out
                ResponseBody = {'Status': 'Internal Error'}                               
                cfnresponse.send(event, context, cfnresponse.FAILED, {ResponseBody}) 

Of notice in this section, this is where we actually attach the provided policy to the provided IAM user group during stack creation.

response = client.attach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                  #we attach the policy to the group  

And this is where we detach it during stack deletion.

response = client.detach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                  #we detach the policy to the group 

In this third part we define the service token needed for the custom resource to work. Please see the following doc for further information [3].

#Custom resource (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html)
  GroupManagedPolicyFunction: 
    Type: Custom::GroupAttachDetach
    Properties:
      ServiceToken: !GetAtt FunctionGroupAttachAndDetach.Arn

In this fourth part, we have created the lambda role. This role will have AWSLambdaBasicExecutionRole [4] permissions along with IAM permission. The IAM permissions are needed to perform the attach and detach operations on the IAM user group.

#Lambda role with Basic Exec permissions (AWSLambdaBasicExecutionRole) and an IAM all permissions, if you create a different custom resource, you should enable lambda to perform related actions
  LambdaCustomResourceRoleGroupAttachAndDetach:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            - logs:CreateLogGroup
            - logs:CreateLogStream
            - logs:PutLogEvents
            - iam:*
            Resource: '*'

## Outputs section

In this part, we defined an outputs section to save the result of the stack state. If successful it will print out the Status as 'Creation Complete'.

Outputs:
  Results:
    Description: Create Result
    Value: !GetAtt GroupManagedPolicyFunction.Status


## Monitoring and debugging

In case a custom resource fails to deploy for whatever reason, CloudFormation will by default delete the lambda that was created to host the custom resource.
To prevent this you will need to select 'Preserve successfully provisioned resources' [4] in the Stack Failure Options during Stack Creation.

This way even if the stack fails you will be able to check the CloudWatch logs for the lambda function.


For this specific example this is the result of the successful deployment.


If you change the RespondeBody to be this:
ResponseBody = response 


Then this is the result of the successful deployment, there are lots of 'customizations' possible for this property.


## References

[1] Code

AWSTemplateFormatVersion: 2010-09-09

#Parameter section, during stack creation I will input an existing IAM Group name
Parameters:
  GroupNameEnv:                             #Existing Group that you input
    Type: String
  ARNAWSManagedPolicyToAttach:              #Any AWS managed policy that you input as ARN
    Type: String

#Resources section, I am creating a custom lambda resource, it will save as environment variable the Group Name that we got from the parameters so that it can be used in the code
Resources:
  FunctionGroupAttachAndDetach:
    Type: 'AWS::Lambda::Function'
    Properties:
      Handler: index.handler
      Role: !GetAtt LambdaCustomResourceRoleGroupAttachAndDetach.Arn
      Environment:
        Variables: 
          GroupNameEnv : !Ref GroupNameEnv
          AWSARNManagedPolicyToAttachEnv: !Ref ARNAWSManagedPolicyToAttach
      Runtime: python3.9
      #Python code
      Code:
        ZipFile: !Sub |
            import os                                                                                                                             #Needed to 'import' the Environment variable
            import boto3                                                                                                                          #Needed to use custom resources
            import cfnresponse                                                                                                                    #Needed to use custom resources        
            GroupNameVar = os.environ['GroupNameEnv']                                                                                             #Importing the Group name and saving it as GroupNameVar
            AWSARNManagedPolicyToAttachVar = os.environ['AWSARNManagedPolicyToAttachEnv']                                                         #Importing the Managed policy arn and saving it as ARNManagedPolicyToAttachVar
            client = boto3.client('iam')                                                                                                          #Importing iam custom resources
            def handler(event, context):
              try:
                if event['RequestType'] == 'Delete':                                                                                              #When in cloudformation we perform delete stack event
                  response = client.detach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                          #we detach the policy to the group 
                  ResponseBody = {'Status': 'Deletion Complete'}                                                                                  #Creating a 'Status' state, so we can use it output (optional)
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, ResponseBody)
                  return
                elif event['RequestType'] == 'Create':                                                                                            #When in cloudformation we perform create stack event
                  response = client.attach_group_policy(GroupName=GroupNameVar,PolicyArn=AWSARNManagedPolicyToAttachVar)                          #We attach the policy to the group  
                  ResponseBody = {'Status': 'Creation Complete'}                                                                                                        #Creating a 'Status' state, so we can use it output (optional)
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, ResponseBody)      
              except Exception:                                                                                                                   #If syntax is wrong, or it fails for any reason, an exception will take us out
                ResponseBody = {'Status': 'Internal Error'}                               
                cfnresponse.send(event, context, cfnresponse.FAILED, {ResponseBody}) 

  #Custom resource (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html)
  GroupManagedPolicyFunction: 
    Type: Custom::GroupAttachDetach
    Properties:
      ServiceToken: !GetAtt FunctionGroupAttachAndDetach.Arn

#Lambda role with Basic Exec permissions (AWSLambdaBasicExecutionRole) and an IAM all permissions, if you create a different custom resource, you should enable lambda to perform related actions
  LambdaCustomResourceRoleGroupAttachAndDetach:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            - logs:CreateLogGroup
            - logs:CreateLogStream
            - logs:PutLogEvents
            - iam:*
            Resource: '*'
`Outputs:
  Results:
    Description: Create Result
    Value: !GetAtt GroupManagedPolicyFunction.Status`

[2] https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/iam/client/attach_group_policy.html

[3] https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html

[4] https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stack-failure-options.html?icmpid=docs_cfn_console



