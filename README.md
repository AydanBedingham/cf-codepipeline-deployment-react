# AWS CodePipeline: ReactJS cross-account deployment pipeline

CloudFormation template that defines a CodePipeline and associated CodeBuild projects to fascilitate deployment of a ReactJS frontend web application from a shared-services account to development and production accounts.
The ReactJS application is deployed to serverless infrastructure backed by CloudFront distribution and S3 Bucket (CloudFront configured to use S3 Origin).

The pipeline was built and tested with the [cf-react-cors-spa](https://github.com/AydanBedingham/cf-react-cors-spa) project and can be adapted to work with other ReactJS projects that follow a similar project structure and package manager.

## Architecture
1. Developer commit code to the 'monitored branch' (default: main) of the CodeRepository.
2. CodePipeline executes in response to the newly committed code.
3. Pipeline retrieves source from CodeCommit Repository.
4. CodeBuild installs required dependencies using the yarn package manager and creates ReactJS production build artifacts.
5. CodeBuild assumes the cross account role in the Development account.
6. CodePipeline creates/updates the CloudFormation stack that defines the AWS resources used to run the ReactJS application.
7. CloudFormation creates/updates the S3 bucket.
8. CloudFormation creates/updates the CloudFront Distribution.
9. CodePipeline syncs ReactJS production build artifacts to the S3 bucket in the Development account.
10. CodePipeline invalidates the cache of the CloudFront distribution (forcing it to serve the latest content from the S3 bucket).
11. Mandatory approval step for deployment to production environment.
12. -> 18. Repeats steps 5 - 10 for the production environment.

![Screenshot](res/ReactJS-CodeDeploy-Pipeline.drawio.svg?raw=true)


## Templates
`cross-account-deployment-role.yml` : CloudFormation template for deploying cross-account roles. This template is intended to be deployed to the development and production accounts.

`pipeline-reactjs.yml` : CloudFormation template that creates a CodePipeline and associated ClouBuild resources for building and deploying the ReactJS Application. This template is intended to be deployed to the shared services account.

`repository.yml` : Helper template used to create a CodeCommit repository to store the ReactJS application for use with the `pipeline-reactjs.yml` template. This template is intended to be deployed to the shared services account.

## Example Deployment

1. Deploy the `cross-account-deployment-role.yml` template into your development and production accounts to create the cross-account roles that will be assumed by the pipeline.
```
ExternalAccountId: <YOUR SHARED SERVICES ACCOUNT ID>
```

2. Deploy the `repository.yml` template into your Shared Services account to create a new repository.
```
RepositoryName: my-repository
```

3. Populate the repository's `main` branch with contents of the [cf-react-cors-spa](https://github.com/AydanBedingham/cf-react-cors-spa) project.

3. Deploy the `pipeline-reactjs.yml` template into your shared services account
```
CodeCommitRepositoryName: my-repository
BuildBranch: main
DevDeploymentCrossAccountRoleArn: arn:aws:iam::<YOUR DEV ACCOUNT ID>:role/crossaccount-deployment-role
ProdDeploymentCrossAccountRoleArn: arn:aws:iam::<YOUR PROD ACCOUNT ID>:role/crossaccount-deployment-role
DestinationStackName: react-cors-spa
CloudFormationFile: react-cors-spa-stack.yaml
DeploymentRegion: us-east-1
```
