+++
title = 'Setup Amazon Cognito Identity for GitHub Actions - Enhanced AuthFlow'
date = 2023-10-24
type = "page"
+++

This guide will create an **Amazon Cognito Identity Pool** that allows GitHub Actions to authenticate using OpenID Connect. We'll be using the **Enhanched (Simplified) AuthFlow**. We will also create a **S3 Bucket** so that we can test the attribute-based access control (ABAC) from the GitHub Actions workflow.

### Note
It's `very important` to limit which entities in GitHub Actions are allowed to assume roles in your AWS Account(s). Misconfiguration in either **AWS IAM Role trust policies** or a **Cognito Identity Pool** could lead to every organization and user in GitHub being allowed access to your AWS resources from GitHub Actions.

`Try everything in a sandbox environment first.`

### Prerequisite
If the AWS Account does not already contain an AWS IAM OpenID Connect Identity Provider, follow this [guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html). Choose `https://token.actions.githubusercontent.com` for URL and `cognito-identity.amazonaws.com` for **Audience**.

If the AWS Account already contains an AWS IAM OpenID Connect Identity Provider, make sure `cognito-identity.amazonaws.com` is one of the audiences.

## Setup Identity Pool

1. Sign in to https://console.aws.amazon.com/cognito/home in your **AWS sandbox account** and select Identity Pools.
    - Make sure it's the new Cognito Console (mid-2023).
2. Choose to create a new Identity Pool in a region of your choice.
3. In step **Configure identity pool trust**, select `Authenticated Access`, `OpenID Connect` and then proceed.
4. In **Configure Permissions**, select **Create a new IAM Role** and enter a role name of your choice.
5. In **Connect Identity Providers**:
    - Select the **GitHub OIDC identity provider**.
    - Select **Use default authenticated role** in Role Settings. We will further configure this after having created the Identity Pool.
    - Select **Use custom mappings** in **claim mappings**
        - Enter `repository` as both tag name and claim.
        - Click **add another** and enter `environment` in both tag name and claim.
6. In **Configure Properties** give the Identity pool a name and then proceed.
7. Create Identity Pool.
8. Navigate into the newly created Identity Pool using the AWS Console, select the **User Access** tab and click on `token.actions.githubusercontent.com` under Identity Providers.
9. Click **Edit** on **Role Settings**.
10. Select **Choose role with rules**.
11. Add a role assignment:
    - Enter `repository_owner` in the field **Claim**.
    - Keep `Equals` option in the **Operator** field.
    - Enter the name of your GitHub **organization** or your GitHub **username** where you intend to run a GitHub Actions when testing this Identity Pool.
    - In the **Role** box, search for the name of the role that you created in `step 4` above and select it.
12. In **Role resolution**, select `Deny Request`.
13. Save.

## Create S3 Bucket and an inline Policy

1. Create a new S3 bucket with any name you like.
   - We create this S3 Bucket so that we can test attribute based access control from GitHub Actions.
2. Go to AWS IAM Console and find the role that was created in `step 4` during **Setup Identity Pool**.
3. Add the policy below as a new inline policy and change `BUCKET_NAME` in the policy to the name of the bucket created in `step 1` and save the policy.
   - These permissions will allow the GitHub Actions workflow to work with files in the bucket under object prefixes matching claims.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::BUCKET_NAME/repo/${aws:principalTag:repository}/*",
        "arn:aws:s3:::BUCKET_NAME/env/${aws:principalTag:repository}/${aws:principalTag:environment}/*"
      ]
    }
  ]
}
```

## Update AWS IAM Trust policy

1. After adding the inline policy above, move over to the trust policy section of the same role.
2. Add `sts:TagSession` as a permission in `Action`.
   - This is required when mapping claims in Cognito Identity.
3. Add `aws:requestTag/repository` as a `StringLike` condition and change `catnekaise` in `catnekaise/*` to the name of your GitHub organization or GitHub user where you are testing this.
   - This will make it so only GitHub Actions running under your GitHub organization or GitHub user can assume this role via Cognito Identity.
4. The trust policy should now look as below, but with values corresponding to your identity pool and GitHub organization/user.

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
          "cognito-identity.amazonaws.com:aud": "eu-west:11111111-example"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        },
        "StringLike": {
          "aws:requestTag/repository": ["catnekaise/*"]
        }
      }
    }
  ]
}
```

## Create a new GitHub repo for testing

1. Create a new private GitHub repository under the organization or user selected in `step 11` during **Setup Identity Pool**.
2. Create a new workflow in the repository, for example `.github/workflows/cognito-testing.yaml`.
3. Copy and paste the workflow below while also changing the following:
    - Change value for the `BUCKET_NAME` env variable to the name of S3 Bucket you created earlier.
    - In the inputs provided to the `catnekaise/cognito-idpool-auth@v1` action, change the following:
      - `cognito-identity-pool-id` to match the ID of the Identity Pool created earlier. The value should look similar to example value.
      - `aws-region` to match the name of region where the Identity Pool was created
      - `aws-account-id` to match the ID of the AWS Account where the Identity Pool was created.
      - Optionally change `audience` if having configured the OpenID Connect Provider with a different audience than `cognito-identity.amazonaws.com`
4. Save the changes and manually **trigger** the workflow under actions in GitHub.
5. If everything was configured correctly, the workflow run should succeed and there should be two text files in the S3 Bucket that was created earlier.
   - One file should be located at `repo/ORG_NAME/REPO_NAME/info.txt`
   - One file should be located at `env/ORG_NAME/REPO_NAME/ENV_NAME/info.txt`
6. Scroll down past the yaml example.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: string
        required: true
        description: "Environment"
        default: "dev"
env:
  BUCKET_NAME: "BUCKET_NAME"
jobs:
  job1:
    runs-on: ubuntu-latest
    # Repository environments can be used in any repository in any GitHub plan,
    # but only public repos or Enterprise plans support settings protection rules for the environment.
    environment: ${{ inputs.environment }}
    permissions:
      id-token: write
    steps:
      - name: Get Credentials from Amazon Cognito Identity
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          audience: "cognito-identity.amazonaws.com"
          aws-account-id: "111111111111"
          set-in-environment: true

      - name: STS Get Caller Identity
        run: |
          aws sts get-caller-identity

      - name: Upload txt file to S3
        run: |
          echo "Hello from GitHub Actions" > info.txt
          aws s3 cp info.text s3://${{ env.BUCKET_NAME }}/repo/${{ github.repository }}/info.txt
          aws s3 cp info.text s3://${{ env.BUCKET_NAME }}/env/${{ github.repository }}/${{ inputs.environment }}/info.txt

```

#### Using GitHub Enterprise?
Your enterprise owner may have restricted which actions can be used inside GitHub Actions for your enterprise/organization. This is fine as securing the supply chain is important.

If you want to proceed, recommendation is that you either head over to [catnekaise/cognito-idpool-auth](https://github.com/catnekaise/cognito-idpool-auth) and press the `use this template` button to create an action repository in your organization that you can use, or copy-paste the `action.yml` of the same repository into your test repository above.

Whichever way you choose you should review the `action.yml` before you use it. Read more [here](https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions#adding-an-action-from-the-same-repository) about how to use actions from within the same repository as the workflow.


## Continue testing
The permissions given to the role will only allow this workflow to work with files at locations in the S3 Bucket that matches claims in the access token.

- If the repository owner (organization or user) has the name `catnekaise` and the repository is named `example-repo`, the workflow will only be able to work with files under the object prefix `/repo/catnekaise/example-repo/`
- In addition to above, if running in the environment named `dev`, the workflow can also work with files under the object prefix `/env/catnekaise/example-repo/dev/`

Change where in the S3 bucket files are put by manually setting values in the workflow that would conflict with the name of the repository or the environment it runs in and see that the workflow will fail.

## Setup - Basic AuthFlow
Setup for the Basic AuthFlow is similar enough not to warrant a completely separate guide.

- On `step 6` in the `Setup Identity Pool`, also select `Activate basic flow`
- Skip `step 8 - 13` in `Setup Identity Pool`
- Use action `catnekaise/cognito-idpool-basic-auth@alpha` instead of `catnekaise/cognito-idpool-auth@alpha`
  - Provide `role-arn` input and specify ARN of AWS IAM role that was created

### Cleanup - GitHub
Keep as is or archive the GitHub repository used for testing if you believe it can be of value later. 

### Cleanup - AWS
Make sure to delete the AWS resources that was created when no longer using them. 

1. Delete the Cognito Identity Pool.
2. Delete the AWS IAM Role that was created for the Identity Pool.
3. Delete the objects in the S3 Bucket and then delete the S3 Bucket.

## Next
Have a look [here](./).


