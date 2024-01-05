## Implementing Serverless Web Application Hosting and Delivery on AWS by David Tucker

- What We'll Be Doing:
- Working With The Course Repository:
    - [GitHub Repository](https://github.com/davidtucker/ps-serverless-app)
- Where We Are Starting:
    - Initial state of the project.
    - Monorepo: A repository that includes all elements for an application, or multiple applications, inside of a single repository.
        - This includes frontend code, backend code, and infrastructure definition.
    - We are using YARN instead of NPM.
        - Root: package.json. Workspaces array. Each workspace has its won package.json.
            ```javascript
                yarn workspace <workspace_name> add <package_name>
                yarn workspace <workspace_name> add <package_name> -D
                yarn add <package_name> -W
            ```
        - Install all dependencies up and down the dependency tree
            ```javascript
                yarn
            ```
        - Project
            ```javascript
                yarn -v
                npm install yarn -g
                yarn
            ```
    - Infrastructure. AWS CDK.
    - Services. All backend microservices.
    - WebApp. Mostly-formed React application. NOTE: Services are currently mocked.
        ```javascript
            cd webapp
            yarn start
            yarn lint
        ```
    - "Other" files: package.json. And the abilitity to lint the entire project.
- Creating And Deploying S3 Buckets:
    - Bootstrapping AWS account for the CDK. Establish the environment variable and set to "1" for advanced features.
        - Deploying the buckets and verifying within the AWS console. Also: AWS Cloud Development Kit: The Big Picture.
        ```javascript
            echo $AWS_PROFILE
        ```
        - Bootstrap AWS CDK.
    - Deploying the intitial state of the infrastructure.
    - Adding in multiple S3 buckets.
        ```javascript
            cd infrastructure
            npx cdk deploy
        ```
    - Navigate to CloudFormation. Ensure ApplicationStack has been deployed. Also, CDKToolkit as a stack, has been deployed.
        - Ensure that the correct region has been selected.
    - Create some S3 Buckets.
        - Hosting Web application:
        - lib/core/storage.ts:
        ```typescript
            import * as cdk from '@aws-cdk/core';
            import * as s3 from '@aws-cdk/aws-s3';
            export class AssetStorage extended cdk.Construct {
                public readonly uploadBucket: s3.IBucket;
                public readonly hostingBucket: s3.IBucket;
                public readonly assetBucket: s3.IBucket;
                constructor(scope: cdk.Contruct, id: string) {
                    super(scope, id);
                    this.uploadBucket = new s3.Bucket(this, 'UploadBucket', {
                        encryption s3.BucketEncryption.S3_MANAGED,
                    });
                    this.hostingBucket = new s3.Bucket(this, 'WebHostingBucket', {
                        encryption s3.BucketEncryption.S3_MANAGED,
                    });
                    this.assetBucket = new s3.Bucket(this, 'AssetBucket', {
                        encryption s3.BucketEncryption.S3_MANAGED,
                    });
                }
            }
        ```
        - index.ts:
        ```typescript
            import * as cdk from '@aws-cdk/core';
            import { AssetStorage } from '/storage';
            export class ApplicationStack extended cdk.Construct {
                public readonly uploadBucket: s3.IBucket;
                public readonly hostingBucket: s3.IBucket;
                public readonly assetBucket: s3.IBucket;
                constructor(scope: cdk.Contruct, id: string, props?: cdk.StackProps) {
                    super(scope, id, props);
                    new AssetStorage(this, 'Storage');
                }
            }
        ```
        - AWS: ApplicationStack. Resources. (Three S3 buckets.)
- Deploying a CloudFront Distribution:
    - Create a CloudFront distribution using the SDK.
    - Configuring a CloudFront distribution for a single-page web application.
    - lib/core/webapp.ts:
        ```typescript
            import * as cdk from '@aws-cdk/core';
            import * as s3 from '@aws-cdk/aws-s3';
            import * as deployment from '@aws-cdk/aws-s3-deployment';
            import * as cloud from '@aws-cdk/aws-cloudfront';
            import { execSync } from 'child_process';
            import * as path from 'path';

            interface WebAppProps {
                hostingBucket: s3.IBucket;
            }

            export class WebApp extended cdk.Construct {
                public readonly dstribution: cloud.CloudFrontWebDistribution
                constructor(scope: cdk.Contruct, id: string, props: WebAppProps) {
                    super(scope, id);
                    const oai = new cloud.OriginAccessIdentity(this, 'WebHostingOAI', {});
                    const cloudfrontProps: any = {
                        originConfigs: [{}]
                    };
                    this.distribution = new cloud.CloudFrontWebDistribution(this, 'AppHostingDistribution', cloudfrontProps);
                    props.hostingBucket.grantRead(oai);
                }
            }
        ```
        - Within properties: Define origin. Configure errors.
        - index.ts:
        ```typescript
            import * as cdk from '@aws-cdk/core';
            import { AssetStorage } from '/storage';
            import { WebApp } from '/webapp';
            export class ApplicationStack extended cdk.Construct {
                public readonly uploadBucket: s3.IBucket;
                public readonly hostingBucket: s3.IBucket;
                public readonly assetBucket: s3.IBucket;
                constructor(scope: cdk.Contruct, id: string, props?: cdk.StackProps) {
                    super(scope, id, props);
                    const storage = new AssetStorage(this, 'Storage');
                    new WebApp(this, 'WebApp', {
                        hostingBucket: storage.hostingBucket;
                    });
                }
            }
        ```
        - Confirm deployment changes. IAM. (Over fifteen minutes to create. Perhaps.)
- Building And Deploying The React Web App:
    - How to build:
    ```javascript
        cd webapp
        yarn build
    ```
    - [Tools](https://github.com/davidtucker/cdk-webapp-tools)
    - lib/core/webapp.ts:
        ```typescript
            import * as tools from 'cdk-webapp-tools';

            interface WebAppProps {
                hostingBucket: s3.IBucket;
                relativeWebAppPath: string;
                baseDirectory: string;
            }

            new tools.WebAppDeployment(this, 'WebAppDeploy', {
                baseDirectory: props.baseDirectory,
                relativeWebAppPath: props.relativeWebAppPath,
                webDistributionPaths: ['/*'],
                buildDirectory: 'build',
                bucket: props.hostingBucket,
                prune: true
            });
            new cdk.CfnOutput(this, 'URL', {
                value: `https://${this.webDistribution.distributionDomainName}/`
            });

        ```
    - prune: delete all of the existing files.
    - index.ts:
        ```typescript
            new WebApp(this, 'WebApp', {
                hostingBucket: storage.hostingBucket;
                baseDirectory: '../',
                relativeWebAppPath: 'webapp',
            });
        ```
    ```javascript
        cd ../infrastructure
        npx cdk deploy
    ```
- Next Steps:
    - 