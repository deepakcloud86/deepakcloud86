AWSTemplateFormatVersion: '2010-09-09'

Parameters:     
  IAMUserName:
    Type: String 
    Description: The IAM User name for a new IAM User that will be used to sync the roles for Azure AD federation
    Default: AzureADAutomationUser
  IAMUserGroupName:
    Type: String
    Description: The IAM Group name for a new IAM Group that the IAMUserName will be placed within. 
    Default: AzureADAutomationGroup
  AzureAdFederationAdminRoleName:
    Type: String
    Description: The name of the role that was previously created within the management account.
    Default: AzureAdFederationAdminRole
  AzureAdFederationAssumeRoleName:
    Type: String
    Description: The name of the role to be created within the application account. This will be the role assumed from the management account.
    Default: AzureAdFederationAssumeRole
  ManagementAccountId:
    Type: String
    Description: The account Id for the management account.

Resources:
  # Only allow assumerole to this role from the AzureAdFederationAdminRole in the Management account
  # Only allow the role to view the iam user secret for the IAMUserName user.
  AzureAdFederationAssumeRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Ref AzureAdFederationAssumeRoleName
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement: 
          - Effect: Allow
            Action: 
              - 'sts:AssumeRole'
            Principal: 
              AWS: 
                - !Sub "arn:aws:iam::${ManagementAccountId}:role/${AzureAdFederationAdminRoleName}"
      Policies:
        # The secretsmanager:ListSecrets action does not support resource-level permissions or condition keys.
        # https://docs.aws.amazon.com/service-authorization/latest/reference/list_awssecretsmanager.html
        - PolicyName: "GetAzureAdIAMUserSecrets"
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - "secretsmanager:GetSecretValue"
                  - "secretsmanager:DescribeSecret"
                Resource: !Ref CFNUserSecret
              - Effect: Allow
                Action: 
                  - "secretsmanager:ListSecrets"
                Resource: "*"

  # Creates the user only allowing iam:ListRoles. This user is used by Azure AD to sync roles for federation.
  CFNUser:
    Type: AWS::IAM::User
    Properties:
      UserName: !Ref IAMUserName
  # The iam:ListRoles action does not support resource-level permissions or condition keys.
  # https://docs.aws.amazon.com/service-authorization/latest/reference/list_identityandaccessmanagement.html
  CFNUserGroupPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: "AzureADAllowIAMListRoles"
      Path: "/"
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - "iam:ListRoles"
            Resource: "*"
  CFNUserGroup:
    DependsOn: CFNUserGroupPolicy
    Type: AWS::IAM::Group
    Properties:
      GroupName: !Ref IAMUserGroupName     
      ManagedPolicyArns:
        - !Sub "arn:aws:iam::${AWS::AccountId}:policy/AzureADAllowIAMListRoles"
  CFNUserToGroup:
    DependsOn:
      - CFNUser
      - CFNUserGroup
    Type: AWS::IAM::UserToGroupAddition
    Properties:
      GroupName: !Ref IAMUserGroupName
      Users:
        - !Ref IAMUserName

  # Create and store the IAM access keys for the User
  CFNKeys:
    Type: AWS::IAM::AccessKey
    Properties:
      UserName: !Ref CFNUser

  CFNUserSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Description: 'These are the secrets for the CFNUser'
      Name: 'AzureADFederation/CFNUserSecretAccessKey'
      SecretString: !Join [ 
        '',
        [
          '{ "AccessKey": "',
          !Ref CFNKeys,
          '", "SecretKey": "',
          !GetAtt [CFNKeys, SecretAccessKey],
          '", "UserName": "',
          !Ref IAMUserName,
          '" }'
          ]
        ]
