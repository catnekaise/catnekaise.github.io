+++
title = "GitHub Actions ABAC in AWS - Detailed Explanation"
type = "page"
date = 2023-10-24
lastmod = 2023-12-28
resource = "true"
+++

This is an article containing contextual/behavioural information related to **GitHub Actions**, **Amazon Cognito Identity**, **AWS IAM**, **AWS STS** and their interaction together. This could help when troubleshooting something that is not working or help you understand how a specific part in all of this works.

This article is not a setup guide nor a starting point. Have a look [here](/github-actions-abac-aws/cognito-identity) if you are just starting out with this topic.

### Note
It's **very important** to limit which entities in GitHub Actions are allowed to assume roles in your AWS Account(s). Misconfiguration in either `AWS IAM Role trust policies` or a `Cognito Identity Pool` could lead to every organization and user in GitHub being allowed access to your AWS resources from GitHub Actions. Try everything in a sandbox environment first.

If you find inaccuracies or have other feedback, please create an issue [here](https://github.com/djonser/blog/issues). Any feedback that leads to corrections/improvements will have their contribution annotated.

## Authentication Flows
> https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html

In the context of GitHub Actions and Cognito Identity, the trust policy in AWS IAM Roles will use `cognito-identity.amazonaws.com` as the trusted federated identity provider instead of `arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com` which is used when authenticating directly with `AWS STS`.

The reason for this is that the **GitHub Actions OIDC access token** will first be exchanged for a **Cognito Identity OIDC access token**, and it will be this new token that is used when performing `AssumeRoleWithWebIdentity`, giving the workflow temporary AWS credentials. Read more about this in detail [here](/github-actions-abac-aws/cognito-identity).

There are two types of authentication flows. **Enhanced (simplified) authflow** and **Basic (classic) authflow**. In the context of GitHub Actions both authentication flows offer advantages and disadvantages. Which one to use will depend on the IAM architecture you want to build.

### Basic (Classic) AuthFlow
When using the Basic AuthFlow, **anyone** that knows the **Cognito Identity Pool ID** and the **AWS account ID** that contains the Identity Pool, both of which are not secrets, will be allowed to exchange a **GitHub Actions OIDC access token** for a **Cognito Identity OIDC access token**. It will not matter if the GitHub organization does not belong to you or if the GitHub user works in your organization or not.

Having this **Cognito Identity OIDC access token** is not the same as having access to your AWS resources. 

In order to get access to those AWS resources there would have to be an AWS IAM Role with a trust policy allowing this token to assume the role and get credentials. It's configuration in the trust policy that will make it so only access tokens that has claims you trust is allowed access to your AWS resources. It ends up working almost the same as [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services), but you can now create trust policies using other claims than just the `sub` claim. A better overview of this [here](/github-actions-abac-aws/cognito-identity).

#### Traits of Basic AuthFlow

- Full control over [AssumeRoleWithWebIdentity](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sts/assume-role-with-web-identity.html).
- Access token can assume roles in other AWS accounts than where the Identity Pool issuing the access token was created.
- Cognito Identity OIDC Access Token is available inside the executing GitHub Actions workflows.
    - The value of the token is automatically masked by GitHub Actions.
- The same Cognito Identity Access Token can be used to assume multiple AWS IAM Roles.

#### Authentication Flow

1. A GitHub Actions workflow executes with permission `id-token` set to `write`.
   - In the context of the [first diagram](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html), the workflow execution represents the `Device`.
2. Workflow uses its ID Token to request an Access Token.
   - In the context of the [first diagram](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html), this represents `Login`.
   - [GitHub docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#updating-your-actions-for-oidc)
3. Workflow uses the access token to request [GetId](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cognito-identity/get-id.html) from the **Cognito Identity Pool** and receives an `IdentityId`.
   - More about these identities below at [Identity Pool Identities](#identity-pool-identities).
4. Workflow uses the IdentityId and access token to request [GetOpenIdToken](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cognito-identity/get-open-id-token.html) from the **Cognito Identity Pool** and receives a **Cognito Identity OIDC Access Token**.
5. Workflow performs [AssumeRoleWithWebIdentity](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sts/assume-role-with-web-identity.html) using the **Cognito Identity OIDC Access Token**, and if the trust policy allows for it the workflow now has temporary AWS Credentials that should be [masked](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-a-log) and then put to use.

### Enhanced (Simplified) AuthFlow
When using the Enhanced AuthFlow, no **Cognito Identity OIDC access token** will be returned to the workflow. Instead, the operations `GetOpenIdToken` and `AssumeRoleWithWebIdentity` are now merged into the single operation [GetCredentialsForIdentity](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cognito-identity/get-credentials-for-identity.html). `GetCredentialsForIdentity` will only return temporary AWS credentials on success and never return the **Cognito Identity OIDC access token**.

Configuration on the **Identity Pool** will determine if a Cognito Identity OIDC access token is issued and configuration on the Identity Pool will also determine what AWS IAM Role will be assumed using this access token.

Compared to the basic authflow, some capabilities are removed from the executing GitHub Actions workflow and the Identity Pool receives greater control over the authentication process.

#### Traits of Enhanced AuthFlow

- No control over AssumeRoleWithWebIdentity.
- IAM Roles must belong to the same AWS Account as the Identity Pool.
- Cognito Identity OIDC Access Token is **not** available in the execution GitHub Actions workflows.
- Configuration allows assigning different AWS IAM Roles based on GitHub Actions access token claims.

#### Authentication Flow
Step 1-3 are exactly the same as in **Basic AuthFlow**.

1. A GitHub Actions workflow executes with permission `id-token` set to `write`.
    - In the context of the [first diagram](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html), the workflow execution represents the `Device`.
2. Workflow uses its ID Token to request an Access Token.
    - In the context of the [first diagram](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html), this represents `Login`.
    - [GitHub docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#updating-your-actions-for-oidc)
3. Workflow uses the access token to request [GetId](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cognito-identity/get-id.html) from the **Cognito Identity Pool** and receives an `IdentityId`.
    - More about these identities below at [Identity Pool Identities](#identity-pool-identities).
4. Workflow uses the IdentityId and GitHub Actions access token to request [GetCredentialsForIdentity](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cognito-identity/get-credentials-for-identity.html) from the **Cognito Identity Pool**.
    - This request will trigger the "auth-configuration" ([Rule Evaluation](#rule-evaluation) and [Trust Policy](#trust-policy)) and if the access token does not have the necessary claims, the process stops here.
5. The request was allowed and the workflow now has temporary AWS Credentials that should be [masked](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-a-log) and then put to use.

### Cognito Access Token Example

```json
{
  "sub": "eu-west-1:22222222-example",
  "aud": "eu-west-1:11111111-example",
  "amr": [
    "authenticated",
    "token.actions.githubusercontent.com",
    "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com:OIDC:repo:catnekaise/example-repo:environment:dev"
  ],
  "https://aws.amazon.com/tags": {
    "principal_tags": {
      "actor": [
        "djonser"
      ],
      "job_workflow_ref": [
        "catnekaise/example-repo.github/workflows/oidc.yaml@refs/heads/main"
      ],
      "repository": [
        "catnekaise/example-repo"
      ],
      "runner_environment": [
        "github-hosted"
      ]
    }
  },
  "iss": "https://cognito-identity.amazonaws.com",
  "https://cognito-identity.amazonaws.com/identity-pool-arn": "arn:aws:cognito-identity:eu-west-1:111111111111:identitypool/eu-west-1:11111111-example",
  "exp": 1234567890,
  "iat": 1234567890
}
```

## Role Selection
Role selection is only available to the **Enhanced (simplified) AuthFlow**. Either `Use default authenticated role` or `Choose role with rules` can be used to give temporary AWS credentials to the workflow.

### Use default authenticated role
When selecting **use default authenticated role**, **anyone** that knows the **Cognito Identity Pool ID** and the **AWS account ID** that contains this Identity Pool, both of which are not secrets, will be allowed to **attempt** `AssumeRoleWithWebIdentity` against this role. It will not matter if the GitHub organization does not belong to you or if the GitHub user works in your organization.

However and similar to the **Basic AuthFlow**, the trust policy on the **AWS IAM Role** can be configured to only allow the assumption when GitHub Actions claims have values you trust.

Using this configuration will limit an Identity Pool to a single IAM Role in AWS.

### Choose Role with Rules
> https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#using-rules-to-assign-roles-to-users

When selecting **Choose Role with Rules**, rules can be created to select a specific role based on the value of any single claim in the GitHub Actions access token. For example, it becomes possible to assign **Role A** if the claim **repository** has a value of `catnekaise/example-repo` and assign **Role B** if the claim **run_id** has a value of `1`. Any claim in the GitHub Actions OIDC access token can be used for matching here. A single **Identity Pool** will allow for up to `25` rules.

The **Role Selection** feature of Cognito Identity is a weaker version of that which is available in AWS IAM Role trust policies. These role selection rules should not be considered to primarily be a security feature but rather a feature that enables us to build IAM architecture with it. Trust policies should always be used to further restrict access to the roles assigned by these rules.

Rules are evaluated in the order they are configured. If a match is found, evaluation stops.

### Role Resolution
When using **Choose Role with Rules** and no rule evaluated to true for the GitHub Actions OIDC access token, role resolution occurs. Role resolution allows you to either assign the **default authenticated role** or to **deny the request**.

### Rules
Rules allow for matching a single claim using types `Equals`, `StartsWith`, `Contains` or `NotEquals`. Wildcards cannot be used when matching.

Here are some example rules using claims and example values from the GitHub Actions access token.

```json
[
  {
    "Claim": "sub",
    "MatchType": "StartsWith",
    "Value": "repo:catnekaise/example-repo:environment:dev",
    "RoleARN": "arn:aws:iam::111111111111:role/GhaCognitoDevRole"
  },
  {
    "Claim": "repository",
    "MatchType": "StartsWith",
    "Value": "catnekaise/infrastructure.",
    "RoleARN": "arn:aws:iam::111111111111:role/GhaCognitoInfraRole"
  },
  {
    "Claim": "repository_owner",
    "MatchType": "Equals",
    "Value": "catnekaise",
    "RoleARN": "arn:aws:iam::111111111111:role/GhaCognito"
  }
]
```

## Rule Evaluation
This is the details of `step 4` in Authentication Flow of Enhanced (Simplified) AuthFlow.

1. Validation of the GitHub Actions access token (iss, aud, signature, expiration).
    - I'm not 100% sure about this but would assume some type of access token validation occurs before evaluating the rules.
2. Rules are evaluated that the specified `Claim` matches `Value` using the specified `MatchType`.
    - If a rule does not match, evaluation moves on to the next rule.
    - If no rules match, request is considered ambiguous, `role resolution` is triggered and either `deny request` or `use default authenticated role` will occur.
    - [Official Docs](https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#using-rules-to-assign-roles-to-users)
3. A rule was matched, a **Cognito Identity OIDC access token** is issued and `AssumeRoleWithWebIdentity` is performed on the Role specified in `RoleARN` using this access token.
    - If no rule was matched but **role resolution** was configured with `use default authenticated role`, then a token is issued and `AssumeRoleWithWebIdentity` will be performed against that role.
4. Temporary AWS Credentials are returned to the workflow that should be [masked](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#masking-a-value-in-a-log) and then put to use.

#### Custom Role Arn
> https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#using-tokens-to-assign-roles-to-users

When multiple rules in the same Identity Pool would match claims in the same GitHub Actions access token, the workflow can choose to include a **custom role arn** with the `GetCredentialsForIdentity` request to make it so **RoleARN** must also match the provided custom role arn.

## Claim Mapping
A Cognito Identity Pool can be configured to map claims from an OIDC access token to principal/session tags. This is what will allow attribute based access control. A claim can be mapped to a tag with a different name than the claim.

While the GitHub Actions access token _only_ has ~24 claims and the Identity Pool configuration allows for selecting 50 claims to be mapped, [limits in AWS IAM/STS](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html#reference_iam-quotas-entity-length) will only allow so much data to be entered into the session policy. Read more about this further down in [Session Limits](#session-limit).

### CloudTrail
If Cognito Identity issues an OIDC access token and this token is used in either the basic or enhanced authflow to perform `AssumeRoleWithWebIdentity`, a CloudTrail event will look like this when the trust policy allowed assumption.

```json
{
  "userIdentity": {
    "type": "WebIdentityUser",
    "principalId": "cognito-identity.amazonaws.com:eu-west-1:11111111-example:eu-west-1:22222222-example",
    "userName": "eu-west-1:22222222-example",
    "identityProvider": "cognito-identity.amazonaws.com"
  },
  "eventSource": "sts.amazonaws.com",
  "eventName": "AssumeRoleWithWebIdentity",
  "awsRegion": "eu-west-1",
  "requestParameters": {
    "principalTags": {
      "actor": "djonser",
      "job_workflow_ref": "catnekaise/example-repo/.github/workflows/test.yaml@refs/heads/main",
      "repository": "catnekaise/example-repo",
      "run_number": "4",
      "environment": "dev",
      "run_attempt": "1",
      "sha": "abcdef1234abcdef1234abcdef1234abcdef1234"
    },
    "roleArn": "arn:aws:iam::111111111111:role/GhaCognito",
    "roleSessionName": "CognitoIdentityCredentials"
  },
  "responseElement": {
    "subjectFromWebIdentityToken": "eu-west-1:11111111-example",
    "assumedRoleUser": {
      "arn": "arn:aws:iam::111111111111:role/GhaCognito/CognitoIdentityCredentials"
    },
    "packedPolicySize": 47,
    "provider": "cognito-identity.amazonaws.com",
    "audience": "eu-west-1:11111111-example"
  }
}
```

If the trust policy don't permit the **Cognito Identity OIDC access token** to assume the role, the error equivalent event will contain the principalTags, allowing you to trace where it was used from.

## AWS IAM Identity Provider
An Identity Provider for `token.actions.githubusercontent.com` must exist in the AWS Account where the Cognito Identity Pool is created for integrating with GitHub Actions. Following the convention of using `sts.amazonaws.com` as the audience when assuming a role from AWS STS, `cognito-identity.amazonaws.com` can be used as the audience when using Cognito Identity.

AWS STS will receive the Cognito Identity OIDC access token where the audience will be the **Cognito Identity Pool ID**.

## Trust Policy
> https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#creating-roles-for-role-mapping

Below is the _default_ trust policy for a role that should be assumed using the Cognito Identity OIDC access token. It's the policy created when asking the AWS Console to create an IAM Role during set up, and it's also the policy example used in official documentation.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity"
      ],
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "eu-west-1:11111111-example"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        }
      }
    }
  ]
}
```

### sts:TagSession
As soon a single claim is mapped in an Identity Pool, `sts:TagSession` must be added as an `Action` in the trust policy. Failing to add this will result in failure to assume this role.

### Service-Linked Role

- When using the AWS Console to create a Cognito Identity Pool and selecting `New IAM Role`, the role becomes service-linked and will have an ARN like `arn:aws:iam::111111111111:role/service-role/GhaCognito`.
  - The service-linked role will be granted `cognito-identity:GetCredentialsForIdentity` for `*` resources. In the context of GitHub Actions getting credentials, the role does not need this permission.
- It's not required that roles assumed via Cognito Identity exist under the role path service-roles.

### cognito-identity.amazonaws.com:amr
> https://docs.aws.amazon.com/cognito/latest/developerguide/role-trust-and-permissions.html

- When using the value `authenticated` for matching `cognito-identity.amazonaws.com:amr` as seen in the trust policy above, it requires an identity authenticated with **any** identity provider attached to the identity pool.
    - A single Identity Pool can be configured with multiple Identity Providers.
- It's also possible to match [amr](https://openid.net/specs/openid-connect-core-1_0.html) with `token.actions.githubusercontent.com` or `arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com:OIDC:repo:catnekaise/example-repo:environment:dev` (`identity_provider_arn:OIDC:value_of_sub_claim`) instead.
- If using any value other than `authenticated` for the role that is configured as the `Default Authenticated Role`, the Cognito Identity Console UI will display a large warning saying "Trust policy isn't secure for this identity pool". This is incorrect and is likely on a TODO list somewhere.

## Conditions
As previously mentioned, more conditions can and should be added to the trust policy of the roles assumed with the Cognito Identity OIDC access token. At the very least a condition that makes sure that the workflow execution is happening within our GitHub organization or enterprise should be added.

Assuming the claim `repository` is mapped to a tag also named `repository`, the example below would only allow `AssumeRoleWithWebIdentity` if the repository executing GitHub Actions is owned by the `catnekaise` organization in GitHub.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "eu-west-1:11111111-example"
        },
        "StringLike": {
          "aws:requestTag/repository": ["catnekaise/*"]
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "token.actions.githubusercontent.com"
        }
      }
    }
  ]
}
```

#### Many conditions
A trust policy has a [soft quota](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html#reference_iam-quotas-entity-length) of 2048 characters, and it can be raised up to 4096 characters.

```json
{
  "Condition": {
    "StringEquals": {
      "cognito-identity.amazonaws.com:aud": "eu-west-1:11111111-example",
      "aws:requestTag/environment": ["prod"],
      "aws:requestTag/runner_environment": ["self-hosted"],
      "aws:requestTag/actor": ["FirstName_LastName_SUF","FirstName2_LastName2_SUF","FirstName3_LastName3_SUF","FirstName4_LastName4_SUF"],
      "aws:requestTag/job_workflow_ref": ["catnekaise/shared-workflows/.github/workflows/deploy.yaml@abcdef1234abcdef1234abcdef1234abcdef1234"]
    },
    "StringLike": {
      "aws:requestTag/repository": ["catnekaise/*"]
    },
    "ForAnyValue:StringLike": {
      "cognito-identity.amazonaws.com:amr": "token.actions.githubusercontent.com"
    }
  }
}
```

When validating claims in the trust policy, any claim can be used during role selection in the Identity Pool as the trust policy is the mechanism for securing access to temporary AWS credentials.

## Tags

- All GitHub Actions access token claims that were mapped are both a `session tag` and `principal tag`.
- Any `resource tag` attached to the `AWS IAM Role` that was assumed is also a `principal tag`.
- Any `session tag` (claim) with the same tag name as a `resource tag` will override the value of the `resource tag`.

It's not uncommon to have a tag describing which environment a resource belongs to, attached to AWS resources. In the context of GitHub Actions, the `environment` claim is only present when running the workflow in a repository environment. Make sure that the tag name used for mapping the `environment` claim does not conflict with a resource tag attached to the AWS IAM role that would describe an environment name as it could lead to unintended consequences. 

A scenario that could happen is:

- The resource tag `environment` is attached to all your AWS resources.
- The AWS IAM Role that will be assumed via the Identity Pool has the `environment` tag with value set to `prod`.
- The Trust Policy of the AWS IAM Role is not configured to check value/presence of `aws:requestTag/environment`
- A GitHub Actions workflow executes with no environment configured on the workflow job.
- No `environment` claim is mapped to a principal tag in Cognito Identity as the claim is not present in the GitHub Actions access token.
- Role is assumed and now have a `principal tag` named `environment` with value `prod` without having been executed in a repository environment.

In short, be mindful of resource tags attached to AWS IAM Roles when using ABAC.

## Role Chaining Reference
> https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html#id_session-tags_role-chaining

### Role Chaining - Trust a role

- Include action `sts:TagSession` if intending to pass existing session tags.
- Use `aws:principalTag/tag_name` to require existing tag (claim) to have specified value.
- Set `aws:requestTag/tag_name` to match `${aws:principalTag/tag_name}` to ensure that tags/claims are not changed when performing `AssumeRole` (if passing existing tags).
- Use `aws:tagKeys` to restrict which session tags can be set.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/GhaCognito"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "aws:principalTag/environment": "prod",
          "aws:requestTag/repository": "${aws:principalTag/repository}",
          "aws:requestTag/job_workflow_ref": "${aws:principalTag/job_workflow_ref}",
          "aws:requestTag/environment": "${aws:principalTag/environment}"
        },
        "ForAllValues:StringEquals": {
          "aws:TagKeys": [
            "repository",
            "environment",
            "job_workflow_ref"
          ]
        },
        "StringLike": {
          "aws:PrincipalTag/repository": [
            "catnekaise/example.*"
          ]
        }
      }
    }
  ]
}
```

#### Role Chaining - GhaCognito Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Resource": [
        "arn:aws:iam::222222222222:role/WorkloadRole"
      ]
    }
  ]
}
```

### Role Chaining - Trust an Entrypoint Account
Use this to trust that a specific entrypoint account have correctly authenticated users via a Cognito Identity Pool.

- Include action `sts:TagSession` if intending to pass existing session tags.
- Use `aws:principalTag/tag_name` to require existing tag (claim) to have specified value.
- Set `aws:requestTag/tag_name` to match `${aws:principalTag/tag_name}` to ensure that tags/claims are not changed when performing `AssumeRole` (if passing existing tags).
- Use `aws:tagKeys` to restrict which session tags can be set.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          // Crucial that the account 111111111111 only uses Cognito Identity for GitHub Actions OIDC and nothing else
          "aws:FederatedProvider": "cognito-identity.amazonaws.com",
          "aws:PrincipalTag/environment": "prod",
          "aws:RequestTag/repository": "${aws:principalTag/repository}",
          "aws:RequestTag/job_workflow_ref": "${aws:principalTag/job_workflow_ref}",
          "aws:RequestTag/environment": "${aws:principalTag/environment}"
        },
        "ForAllValues:StringEquals": {
          "aws:TagKeys": [
            "repository",
            "environment",
            "job_workflow_ref"
          ]
        },
        "StringLike": {
          "aws:PrincipalTag/repository": [
            "catnekaise/example.*"
          ]
        }
      }
    }
  ]
}
```

##### Role Chaining - Alternative for Permissions
Granting the permissions below to roles in an entrypoint account makes it so that no changes are required when additional workload accounts are integrated. The conditions are used so prevent other roles that trusts the same entrypoint account from being assumable.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      // Include when passing claims
      "Effect": "Allow",
      "Action": "sts:TagSession",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      // Role Path to limit assuming roles to those under this path
      "Resource": "arn:aws:iam::*:role/github-actions/*",
      "Condition": {
        "StringEquals": {
          // ResourceTag to limit that roles being assumed have this tag
          "aws:ResourceTag/tag1": "someValue",
          // Shall not be able to assume roles in a different organization
          "aws:ResourceOrgID": "o-example"
        },
        "StringNotEquals": {
          // Shall not be allowed to assume roles in these accounts
          "aws:ResourceAccount": [
            "111111111111"
          ]
        },
        "ForAnyValue:StringLike": {
          // Optionally limit to accounts in organization path(s) below
          "aws:ResourceOrgPaths": [
            "o-example/r-ab12/ou-ab12-11111111/*"
          ]
        }
      }
    }
  ]
}
```

##### Assume Role
Inside a GitHub Actions workflow it would look similar to this when using the AWS CLI directly.

```shell
aws sts assume-role \
--role-arn arn:aws:iam::222222222222:role/chained-gha-role \
--role-session-name chained \
--tags Key=environment,Value='${{ inputs.environment }}' Key=repository,Value='${{ github.repository }}'
```

### Role Chaining CloudTrail Event
`sts:AssumeRole` using the credentials provided from Cognito Identity will include the information below in the CloudTrail event.

The `sub` claim is added as a suffix on identity provider arn in `cognito-identity.amazonaws.com:amr`.

```json
{
  "userIdentity": {
    "webIdFederationData": {
      "federatedProvider": "cognito-identity.amazonaws.com",
      "attributes": {
        "cognito-identity.amazonaws.com:amr": "[\"authenticated\",\"token.actions.githubusercontent.com\",\"arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com:OIDC:repo:catnekaise/example-repo:environment:dev\"]",
        "cognito-identity.amazonaws.com:aud": "eu-west-1:11111111-example",
        "cognito-identity.amazonaws.com:sub": "eu-west-1:22222222-example"
      }
    }
  }
}
```

## Permission Examples

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/${aws:principalTag/repository}/${aws:principalTag/sha}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/build-tools/*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalTag/job_workflow_ref": "catnekaise/shared-workflows/.github/workflows/*@refs/heads/main"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem"
      ],
      "Resource": "arn:aws:dynamodb:eu-west-1:111111111111:table/my-table",
      "Condition": {
        "ForAllValues:StringLike": {
          "dynamodb:LeadingKeys": [
            "${aws:principalTag/repository}/${aws:principalTag/environment}/*"
          ]
        }
      }
    }
  ]
}
```

## S3 Bucket Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotPrincipal": {
        "AWS": "arn:aws:iam::111111111111:role/GhaCognito"
      },
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringNotLike": {
          "aws:requestTag/job_workflow_ref": ["catnekaise/shared-workflows/.github/workflows/*@refs/heads/main"]
        }
      }
    }
  ]
}
```

## Identity Pool Identities

- Any GitHub Actions workflow executed by any GitHub user can create an identity with a [Cognito Identity Pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity.html) configured with GitHub Actions as an Identity Provider.
    - This is as long as the workflow has the combination of the identity pool id (`eu-west-1:11111111-example`) and the AWS account ID hosting the identity pool, both of which are not secrets.
    - No workflow can get temporary AWS credentials unless the identity pool configuration and AWS IAM Role trust policy allows for it.
- One Identity is created in the Identity Pool per unique `sub` claim.
    - `sub` claim `repo:catnekaise/example-repo:environment:dev` will have a different `IdentityId` than `repo:catnekaise/example-repo:environment:prod`
    - Depending on [customization](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims) of the `sub` claim, each workflow run has the potential to create a new identity in the pool.
    - There's no quota on identities per identity pool.

## Quotas
> https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html#amazon-cognito-identity-pools-federated-identities-request-rate-quotas

#### Adjustable
- `GetOpenIdToken` - 200 Request Per Second (Basic AuthFlow)
- `GetCredentialsForIdentity` - 200 RPS (Enhanced AuthFlow)
- `GetId` - 25 RPS (Basic and Enhanced AuthFlow)
    - Since GetId is less than both `GetOpenIdToken` and `GetCredentialsForIdentity` it cancels out those quotas.
    - The reason for the discrepancy between 25 and 200 is as the [documentation states](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html): "The application is expected to cache this identity ID to make subsequent calls to Amazon Cognito".

#### Non-Adjustable
- `Role-based access control (RBAC) rules` - 25 rules per Identity Pool

## Session Limit
Sessions are involved when using `AssumeRoleWithWebIdentity`. Read more [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html#reference_iam-quotas-entity-length) and [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_session).

Long story short (read official docs), if there's too much input during `AssumeRoleWithWebIdentity` it could fail due to `PackedPolicyTooLargeException`. In order to avoid this, here are aspects that can be controlled from a higher degree to a low degree:

- The number of GitHub Actions access token claims mapped.
- The tag names selected for the mapped claims.
- The size of the [customized](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims) `sub` claim.
- Value of some claims.

#### PackedPolicyTooLargeException
It should also be noted that in this particular context the `packedPolicySize` property of `responseElements` in CloudTrail (see further above) can be misleading. It's possible to be below 50% of allotted space and just be a few bytes away from receiving `PackedPolicyTooLargeException` with the message `Serialized token too large for session`.

```json
{
  "errorCode": "PackedPolicyTooLargeException",
  "errorMessage": "Packed size of session tags consumes 138% of allotted space."
}
```

```json
{
  "errorCode": "PackedPolicyTooLargeException",
  "errorMessage": "Serialized token too large for session"
}
```

### Managing the Session Limit

- Information included in `repository_owner` is already present in the `repository` claim. Skip mapping `repository_owner`.
- If using GitHub Enterprise, don't map the `enterprise` claim, instead [customize the issuer](https://docs.github.com/en/enterprise-cloud@latest/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-issuer-value-for-an-enterprise).
- Don't map the `workflow` claim as it's a string in the workflow file that can grow large.
- Abbreviate all tag names.
    - For example, `runEnv` instead of `runner_environment`, `jWorkRef` instead of `job_workflow_ref`, `run` instead of `run_number`.
    - Don't shorten tag names into acronyms or single character tag names as it makes it harder to use these tags in policies and cloudtrail logs.
- Consider not [customizing](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims) the `sub` claim.
    - The reason for this is that the value of the `sub` claim is included in `cognito-identity.amazonaws.com:amr` which takes up space in the session.
    - If using Cognito Identity in GitHub Actions while also requiring an existing AWS STS authentication integration to exist in parallell where the current `sub` claim has been customized and made "large", then it will affect how many claims can be mapped.
- If mapping the `job_workflow_ref` claim, consider having a policy for workflow filenames so that filenames never end up looking like `deployment-infrastructure-terraform-self-hosted-github-actions-only-v1-2023.yaml`. This will save bytes.
- `job_workflow_ref` is more important than `workflow_ref`.

If not having customized the `sub` claim, then it should be possible to map the following claim when using tag name abbreviations: `repository`, `environment`, `job_workflow_ref`, `actor`, `runner_environment`, `run_number`, `run_attempt`, `ref` and `sha`, granted names of organization, repositories and workflow filenames are... average(?).

All available GitHub Actions claims can be viewed [here](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token).

##### Testing
It can be beneficial to test the limits out. Do so by creating a repo and workflow file both with long names and an Identity Pool for testing. When everything seems fine and all required claims fit, start adding more claims to see when the limit is hit.

The goal is to have plenty of room to spare and never having to deal with `PackedPolicyTooLargeException`.

## Resources

#### Additional AWS Documentation

- https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_attribute-based-access-control.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub