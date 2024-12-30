# Scalable Moodle Deployment on AWS ECS Fargate

This project provides an AWS CDK application for deploying a scalable Moodle installation using Amazon ECS Fargate, RDS, ElastiCache, and CloudFront.


## Architecture

![alt text](Architecture-v1.drawio.png)

## Project Description

This AWS CDK application automates the deployment of a highly available and scalable Moodle learning management system. It leverages various AWS services to create a robust infrastructure:

- Amazon ECS Fargate for running Moodle containers
- Amazon RDS for MySQL database
- Amazon ElastiCache for Redis caching
- Amazon EFS for shared file storage
- Application Load Balancer for traffic distribution
- Amazon CloudFront for content delivery
- AWS WAF for web application firewall protection

The infrastructure is designed with security and scalability in mind, utilizing VPC endpoints, security groups, and auto-scaling capabilities. It also includes CloudTrail for auditing and SNS notifications for RDS events.

## Repository Structure

```
.
├── bin
│   └── cdk.ts                 # Entry point for the CDK application
├── lib
│   ├── cloudfront-waf-web-acl-stack.ts  # CloudFront WAF Web ACL stack
│   ├── ecs-moodle-stack.ts    # Main ECS Moodle stack
│   └── ssm-parameter-reader.ts  # Utility for reading SSM parameters
├── src
│   └── image
│       └── src
│           ├── Dockerfile     # Dockerfile for Moodle image
│           └── libmoodle.sh   # Moodle configuration script
├── cdk.json                   # CDK configuration
├── package.json               # Node.js dependencies
└── tsconfig.json              # TypeScript configuration
```

## Usage Instructions



### Prerequisites

1. [Install and configure AWS Command Line Interface (AWS CLI) with your AWS Identity and Access Management (IAM) user](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).

2. [Install AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_install).

3. [Install Docker](https://docs.docker.com/engine/install/).

4. [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git).

5\. Pull the source code into your machine by running the following in your terminal:git clone https://github.com/aws-samples/aws-cdk-ecs-refarch-moodle.git

### Setting up domain name and TLS certificate

1\. Set up a public domain name to request a public certificate in ACM. If you don’t have a public domain name yet, you can use [Amazon Route 53](https://aws.amazon.com/route53/) to [register a new domain](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html). This domain name is also used for CloudFront alternative domain name.

2. [Request two public certificates](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) for your domain name using ACM. The first one is for the ALB where this solution is deployed (e.g. ap-southeast-1); the second one is for the CloudFront in the us-east-1 Region. For example: moodle.example.com or \*.example.com. Note the certificate Amazon Resource Names (ARNs) to be used in the deployment steps.

### Publishing Moodle container image to Amazon ECR

Prior to deploying the solution, you must first build the Moodle container image locally and publish it into a container registry such as Amazon ECR or an external registry of your choice. In this case, I use Amazon ECR.

1\. Open your preferred command line interface (CLI) such as Terminal or Command Prompt.

2\. From the top directory of the source code, run the following to build the container image:

```
docker build -t moodle-image src/image/src
``` 
or 
```
docker pull bitnami/moodle:latest
```

3\. Authenticate to your default AWS account registry:
```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 024848486969.dkr.ecr.us-east-1.amazonaws.com
```

4\. Create a new ECR Repository to hold the image:
```
aws ecr create-repository --repository-name moodle-image --region us-east-1
```

5\. Tag the image to push to your repository:
```
docker tag moodle-image:latest 024848486969.dkr.ecr.us-east-1.amazonaws.com/moodle-image:latest
```
or 
```
docker tag bitnami/moodle:latest 024848486969.dkr.ecr.us-east-1.amazonaws.com/moodle-image:latest
```


6\. Push the image:
```
docker push 024848486969.dkr.ecr.us-east-1.amazonaws.com/moodle-image:latest
```


### Prerequisites

- Node.js (v14.x or later)
- AWS CLI configured with appropriate credentials
- AWS CDK CLI (v2.x)

### Installation

1. Clone the repository:
   ```
   git clone <repository-url>
   cd <repository-directory>
   ```

2. Install dependencies:
   ```
   npm install
   ```

3. Configure the `cdk.json` file with your specific settings, including:
   - ALB and CloudFront certificate ARNs
   - CloudFront domain
   - Moodle Docker image URI
   - Service replica count
   - RDS and ElastiCache instance types




### Configuration

The main configuration options are set in the `cdk.json` file. You can adjust the following parameters:

- `albCertificateArn`: ARN of the SSL certificate for the ALB
- `cfCertificateArn`: ARN of the SSL certificate for CloudFront
- `cfDomain`: Domain name for the CloudFront distribution
- `moodleImageUri`: URI of the Moodle Docker image
- `serviceReplicaDesiredCount`: Desired number of ECS tasks
- `rdsInstanceType`: RDS instance type
- `elastiCacheRedisInstanceType`: ElastiCache Redis instance type



### Deployment

1. Synthesize the CloudFormation template:
   ```
   cdk synth
   ```

2. Deploy the stacks:
   ```
   cdk deploy --all
   ```

### Troubleshooting

1. ECS Task Failures:
   - Check CloudWatch Logs for the ECS tasks
   - Ensure the Moodle image is accessible and correctly configured
   - Verify that the database connection details are correct

2. Database Connection Issues:
   - Check the security group rules for the RDS instance
   - Verify the database credentials in Secrets Manager

3. CloudFront Access:
   - Ensure the CloudFront distribution is deployed and enabled
   - Check the WAF rules if content is being blocked

For detailed logs and debugging:
- Enable verbose logging in the Moodle container by setting `BITNAMI_DEBUG=true`
- Check CloudWatch Logs for ECS tasks, RDS, and ALB access logs
- Use AWS X-Ray for tracing requests through the application

## Data Flow

The request data flow through the application is as follows:

1. User requests are first received by CloudFront
2. CloudFront forwards requests to the Application Load Balancer
3. The ALB distributes requests to ECS Fargate tasks running Moodle
4. Moodle containers process requests, interacting with:
   - RDS MySQL for database operations
   - ElastiCache Redis for caching
   - EFS for shared file storage
5. Responses flow back through the ALB and CloudFront to the user

```
[User] <-> [CloudFront] <-> [ALB] <-> [ECS Fargate (Moodle)]
                                       |
                                       ├─ [RDS MySQL]
                                       ├─ [ElastiCache Redis]
                                       └─ [EFS]
```

Note: The WAF is integrated with CloudFront to provide an additional layer of security.

## Infrastructure

The main infrastructure components defined in the CDK stacks include:

- VPC:
  * 2 public subnets and 2 private subnets
  * NAT Gateways for outbound internet access from private subnets
  * VPC Endpoints for ECR and S3

- ECS:
  * Fargate cluster
  * Task definition for Moodle container
  * ECS Service with auto-scaling

- RDS:
  * MySQL instance for Moodle database
  * Event subscription for notifications

- ElastiCache:
  * Redis instance for caching

- EFS:
  * File system for shared Moodle data

- Application Load Balancer:
  * Distributes traffic to ECS tasks

- CloudFront:
  * Content delivery network

- WAF:
  * Web Application Firewall integrated with CloudFront

- CloudTrail:
  * Audit logging to S3 bucket

- IAM:
  * Roles and policies for ECS tasks and other resources