**IAM** (Identity and Access Management) is a *global service* meaning that it is not associated with any particular AWS region. IAM manages the **root account** (the one you logged in with) which is created by default. This information should NOT be shared. We should create **users** right away and ***we should not use the root user, as is best practice***.

#### Users, Groups, and Policies

**Users** are people within the organization which can be in **groups**. Groups only contain users, they cannot contain other groups. Users are also allowed to belong to multiple groups, or no groups, though users not being in groups in not best practice. For best practice, one user should be created for one physical person, and IAM user accounts should not be shared.

![[images/iam/users-and-groups.png]]


The purpose of creating users and groups is to allow them to use our AWS accounts. But to have multiple users using the same account, we need to give them **permissions**. We do this be assigning a JSON document called a **policy** that describes what users and groups are allowed to do. 

We should not allow all users to do anything within our AWS account, or else they could waste a lot of money, or it might be a security issue. We apply the **least privilege principle**: don't give the user more permissions than the bare minimum they need.

Policies have the schema:
- Version: policy language version usually `2012-10-17`
- Id: an identifier for the policy (optional)
- Statement: array of one or more statements
	- Sid: an identifier for the statement (optional)
	- Effect: whether the statement allows or denies access to certain APIs. Must be either `Allow` or `Deny`
	- Principal: account/user/role to which this policy is applied
	- Action: list of actions this policy allows or denies
	- Resource: list of resources to which the actions apply to
	- Condition: for when this policy is in effect (optional)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "elasticloadbalancing:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

Creating a user:

![[images/iam/creating-a-user.png]]

We can create an admin user with admin access to allow full permission to all AWS Services. It is recommended we instead create groups, and add the policies to the groups. So we create a group called `admin` with one of the OOTB policies `AdministratorAccess` which looks like:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

![[images/iam/assign-group-with-policy.png]]Now the group has `AdministratorAccess` policy, and user `tyler` inherited it too from the `admin` group. If a policy is only associated with a single user or group, it is an **inline policy**, which means if the user or group is deleted so is the policy. Alternatively, a **managed policy** is standalone, it is created separately and can then be attached to users and groups.

![[policy-inheritance.png]]

Now we will want to be able to log in with our users. There's some information to be aware of for signing in to users, The root user has an **Account ID**, and a **sign in URL** for all IAM users that stem from the root user. We customize this url from `https://<AccountID>.signin.aws.amazon.com/console` to `https://<Account Alias>.signin.aws.amazon.com/console` by creating an **account alias**. This is why when you first went to sign in, there was the option to sign in as either a root account or with an account alias. 

![[images/iam/created-user.png]]

> Tip: In order to conveniently log in to multiple users in one browser without needing to open an incognito window, use **multi session support** which will open a separate window to log into another account with a unique session.

![[images/iam/session-metadata.png]]

Let's log in to our new IAM user `tyler`, and go to IAM within it. We could see the user `tyler` in this IAM user because it inherits the `AdministratorAccess` policy, but if we remove it from the `admin` group it will no longer be able to any more.

![[images/iam/access-denied.png]]

We could then give it permission to just this if we don't want `tyler` to be a member of `admin` with the `IAMReadOnlyAccess`, an OOTB Managed policy.

#### Protecting IAM Accounts

In IAM, we can customize our own password policies for creating new users. These include:
- min password length
- require specific char types in the password
- allow all IAM users to change their own passwords
- require users to change their passwords after some time
- prevent password re-use

![[images/iam/password-policy.png]]

Another useful protection mechanism is **Multi-Factor Authentication**. At least the root account, and ideally all IAM users should have MFA for good security practices. MFA is linked to a specific MFA device. There are several options for MFA
- As an app: virtual MFA device (Google Authenticator, Authy, Duo)
- As a flash drive: universal 2nd factor (U2F) Security Key (YubiKey)
- As standalone hardware: Hardware Key Fob (Gemalto, SurePassID)


#### Alternate ways to Access AWS

Besides the AWS Management console, protected by password + MFA, there are other ways to access AWS
- **AWS SDK**: AWS APIs in application code, available in many languages, including mobile and IoT systems.
- **AWS CLI**: Interact with AWS services from your command line shell via public APIs. Used to develop scripts to manage resources. Build on the AWS SDK for python.

In order to use the AWS CLI or SDK, we need to generate an **access key** through the management console. These are secret just like a password, do not share them, but they come with an **access key ID** which is not secret. Like most keys, they are only shown once when generated, never commit them in any public repo. Click on your username in the top right, and go to *security credentials*, which goes to an IAM page that is not accessible from the side menu otherwise.

![[images/iam/security-credentials.png]]

Then we can set up the AWS CLI with `aws configure`, which will ask for our access key, access key ID, default region name, and output format (JSON (default), YAML, text, table).
Now we can try out the CLI with `aws iam list-users` and we get a list of users (not including the root). This is the same API called IAM when we go to the users tab in the side bar menu.

```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "tyler",
            "UserId": "AIDAQYEI5HUZJZ7TTD2TA",
            "Arn": "arn:aws:iam::051826736434:user/tyler",
            "CreateDate": "2026-02-12T01:55:49+00:00",
            "PasswordLastUsed": "2026-02-12T16:17:01+00:00"
        }
    ]
}
```

You can alternatively use **CloudShell** to run a CLI in the cloud, which automatically gets credentials and region from the account you invoked CloudShell from in the management console. Using the `--region` flag, you can also pass in a different region. Any files created within the CloudShell environment are persistent across CloudShell sessions, and files can be uploaded or downloaded

![[images/iam/cloudshell.png]]

#### Roles

Some AWS services need to perform actions on your behalf, so they require permissions too just like users. You grant those permissions by creating an IAM role and attaching policies to it. The service then assumes that role to obtain temporary credentials. Lets create a role for EC2 ahead of the next section of this course on EC2, with the `IAMReadOnlyAccess` policy.

![[images/iam/creating-a-role.png]]

#### Security Tools

IAM has a few different security/auditing tools we can use to review its state.

**IAM Credentials Report**: an account-level security tool to create a report listing all the account's users and the status of their various credentials. 

> Note: there is more data not shown here
![[images/iam/credentials-report.png]]

**IAM Last Accessed:** a user-level security tool that shows service permissions granted to a user and when those services were last accessed. This can be used to revise policies, and help to enforce the principle of least privilege

![[images/iam/last-accessed.png]]