+++
title = "Catnekaise"
type = "page"
date = 2023-10-24
+++

## GitHub Actions ABAC in AWS

- [Cognito Identity in GitHub Actions](./github-actions-abac-aws/cognito-identity) - Start here
- [Setup Cognito Identity using AWS Console](./github-actions-abac-aws/setup-using-aws-console)
- [GitHub Actions ABAC in AWS - Detailed Explanation](./github-actions-abac-aws/detailed-explanation) - Use this to troubleshoot or if you are interested in details
- [catnekaise/actions-constructs](https://github.com/catnekaise/actions-constructs) - AWS CDK constructs purpose-built for GitHub Actions and this use case
- [catnekaise/cognito-idpool-auth](https://github.com/catnekaise/cognito-idpool-auth) - GitHub Action for Cognito Identity Authentication
- [catnekaise/cognito-idpool-basic-auth](https://github.com/catnekaise/cognito-idpool-basic-auth) - GitHub Action for Cognito Identity Authentication - Basic (Classic) AuthFlow

## ghrawel
Use ghrawel to deploy an AWS API Gateway RestAPI capable of returning GitHub App installation access tokens and use AWS IAM to control access to this API.

- [Developer Notes](./ghrawel/developer-notes)
- [catnekaise/ghrawel](https://github.com/catnekaise/ghrawel) - Main repository and CDK library.
- [catnekaise/ghrawel-token](https://github.com/catnekaise/ghrawel-token) - Re-usable action for requesting a token when running in GitHub Actions.
- [catnekaise/ghrawel-tokenprovider-lambda-go](https://github.com/catnekaise/ghrawel-tokenprovider-lambda-go) - Application running in lambda that creates token via GitHub API based on request.