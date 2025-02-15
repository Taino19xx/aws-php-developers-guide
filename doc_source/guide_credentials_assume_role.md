# Assuming IAM roles<a name="guide_credentials_assume_role"></a>

## Using IAM roles for Amazon EC2 instance variable credentials<a name="instance-profile-credentials"></a>

If you’re running your application on an Amazon EC2 instance, the preferred way to provide credentials to make calls to AWS is to use an [IAM role](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) to get temporary security credentials\.

When you use IAM roles, you don’t need to worry about credential management from your application\. They allow an instance to “assume” a role by retrieving temporary credentials from the Amazon EC2 instance’s metadata server\.

The temporary credentials, often referred to as **instance profile credentials**, allow access to the actions and resources that the role’s policy allows\. Amazon EC2 handles all the legwork of securely authenticating instances to the IAM service to assume the role, and periodically refreshing the retrieved role credentials\. This keeps your application secure with almost no work on your part\. For a list of the services that accept temporary security credentials, see [AWS services that work with IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html) in the *IAM User Guide*\.

**Note**  
To avoid hitting the metadata service every time, you can pass an instance of `Aws\CacheInterface` in as the `'credentials'` option to a client constructor\. This lets the SDK use cached instance profile credentials instead\. For details, see [Configuration for the AWS SDK for PHP Version 3](guide_configuration.md)\.

For more information on developing Amazon EC2 applications using the SDKs, see [Using IAM roles for Amazon EC2 instances](https://docs.aws.amazon.com/sdkref/latest/guide/access-iam-roles-for-ec2.html) in the *AWS SDKs and Tools Reference Guide*\.

### Create and assign an IAM role to an Amazon EC2 instance<a name="create-and-assign-an-iam-role-to-an-ec2-instance"></a>

1. Create an IAM client\.

    **Imports** 

   ```
   require 'vendor/autoload.php';
   
   use Aws\Iam\IamClient;
   ```

    **Sample Code** 

   ```
   $client = new IamClient([
       'region' => 'us-west-2',
       'version' => '2010-05-08'
   ]);
   ```

1. Create an IAM role with the permissions for the actions and resources you’ll use\.

    **Sample Code** 

   ```
   $result = $client->createRole([
       'AssumeRolePolicyDocument' => 'IAM JSON Policy', // REQUIRED
       'Description' => 'Description of Role',
       'RoleName' => 'RoleName', // REQUIRED
   ]);
   ```

1. Create an IAM instance profile and store the Amazon Resource Name \(ARN\) from the result\.
**Note**  
If you use the IAM console instead of the AWS SDK for PHP, the console creates an instance profile automatically and gives it the same name as the role to which it corresponds\.  
 **Sample Code**   

   ```
   $IPN = 'InstanceProfileName';
   
   $result = $client->createInstanceProfile([
       'InstanceProfileName' => $IPN ,
   ]);
   
   $ARN = $result['Arn'];
   $InstanceID =  $result['InstanceProfileId'];
   ```

1. Create an Amazon EC2 client\.

    **Imports** 

   ```
   require 'vendor/autoload.php';
   
   use Aws\Ec2\Ec2Client;
   ```

    **Sample Code** 

   ```
   $ec2Client = new Ec2Client([
       'region' => 'us-west-2',
       'version' => '2016-11-15',
   ]);
   ```

1. Add the instance profile to a running or stopped Amazon EC2 instance\. Use the instance profile name of your IAM role\.

    **Sample Code** 

   ```
    $result = $ec2Client->associateIamInstanceProfile([
       'IamInstanceProfile' => [
           'Arn' => $ARN,
           'Name' => $IPN,
       ],
       'InstanceId' => $InstanceID
   ]);
   ```

For more information, see [IAM Roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)\.

## Using IAM roles for Amazon ECS tasks<a name="ecs-credentials"></a>

By using IAM roles for Amazon Elastic Container Service \(Amazon ECS\) tasks, you can specify an IAM role that the containers in a task can use\. This is a strategy for managing credentials for your applications to use, similar to the way that Amazon EC2 instance profiles provide credentials to Amazon EC2 instances\.

Instead of creating and distributing your AWS credentials to the containers or using the Amazon EC2 instance’s role, you can associate an IAM role with an ECS task definition or `RunTask` [API](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-ecs-2014-11-13.html#runtask) operation\.

 For a list of the services that accept temporary security credentials, see [AWS services that work with IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html) in the *IAM User Guide*\.

## Assuming an IAM role in another AWS account<a name="assuming-an-iam-role-in-another-aws-account"></a>

When you work in an AWS account \(Account A\) and want to assume a role in another account \(Account B\), you must first create an IAM role in Account B\. This role allows entities in your account \(Account A\) to perform specific actions in Account B\. For more information about cross\-account access, see [Tutorial: Delegate Access Across AWS Accounts Using IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)\.

After you create a role in Account B, record the Role ARN\. You will use this ARN when you assume the role from Account A\. You assume the role using the AWS credentials associated with your entity in Account A\.

Create an AWS STS client with credentials for your AWS account\. In the following, we used a credentials profile, but you can use any method\. With the newly created AWS STS client, call assume\-role and provide a custom sessionName\. Retrieve the new temporary credentials from the result\. By default credentials last an hour\.

 **Sample Code** 

```
$stsClient = new Aws\Sts\StsClient([
    'profile' => 'default',
    'region' => 'us-east-2',
    'version' => '2011-06-15'
]);

$ARN = "arn:aws:iam::123456789012:role/xaccounts3access";
$sessionName = "s3-access-example";

$result = $stsClient->AssumeRole([
      'RoleArn' => $ARN,
      'RoleSessionName' => $sessionName,
]);

 $s3Client = new S3Client([
    'version'     => '2006-03-01',
    'region'      => 'us-west-2',
    'credentials' =>  [
        'key'    => $result['Credentials']['AccessKeyId'],
        'secret' => $result['Credentials']['SecretAccessKey'],
        'token'  => $result['Credentials']['SessionToken']
    ]
]);
```

For more information, see [Using IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html) or [AssumeRole](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sts-2011-06-15.html#assumerole) in the AWS SDK for PHP API Reference\.

## Using an IAM role with web identity<a name="using-an-iam-role-with-web-identity"></a>

Web Identity Federation enables customers to use third\-party identity providers for authentication when accessing AWS resources\. Before you can assume a role with web identity, you must create an IAM role and configure a web identity provider \(IdP\)\. For more information, see [Creating a Role for Web Identity or OpenID Connect Federation \(Console\)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html)\.

After [creating an identity provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) and [creating a role for your web identity](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html), use an AWS STS client to authenticate a user\. Provide the webIdentityToken and ProviderId for your identity, and the Role ARN for the IAM role with permissions for the user\.

 **Sample Code** 

```
$stsClient = new Aws\Sts\StsClient([
    'profile' => 'default',
    'region' => 'us-east-2',
    'version' => '2011-06-15'
]);

$ARN = "arn:aws:iam::123456789012:role/xaccounts3access";
$sessionName = "s3-access-example";
$duration = 3600;

$result = $stsClient->AssumeRoleWithWebIdentity([
      'WebIdentityToken' => "FACEBOOK_ACCESS_TOKEN",
      'ProviderId' => "graph.facebook.com",
      'RoleArn' => $ARN,
      'RoleSessionName' => $sessionName,
]);

 $s3Client = new S3Client([
    'version'     => '2006-03-01',
    'region'      => 'us-west-2',
    'credentials' =>  [
        'key'    => $result['Credentials']['AccessKeyId'],
        'secret' => $result['Credentials']['SecretAccessKey'],
        'token'  => $result['Credentials']['SessionToken']
    ]
]);
```

For more information, see [AssumeRoleWithWebIdentity—Federation Through a Web\-based Identity Provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html#api_assumerolewithwebidentity.html) or [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sts-2011-06-15.html#assumerolewithwebidentity) in the AWS SDK for PHP API Reference\.