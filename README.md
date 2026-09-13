# CI-CD_Ansible-automation



Yes. First, one important clarification: this is not Ansible code itself. 
It is a GitHub Actions CI/CD workflow that uses Docker to run Ansible Lint. 
Its purpose is to automatically check Ansible code whenever someone pushes to the repository.

1. Overall flow

The workflow does this:

Developer pushes code
        ↓
GitHub Actions starts
        ↓
Find a matching self-hosted runner
        ↓
Checkout repository
        ↓
Configure AWS credentials
        ↓
Login to Amazon ECR
        ↓
Build Ansible Lint Docker image
        ↓
Run ansible-lint inside Docker
        ↓
Pass / Fail the pipeline
2. Workflow name
name: CI/CD automation

This gives the GitHub Actions workflow the name:

CI/CD automation

You'll see this name under the repository's Actions tab.

3. When does the workflow run?
on:
  push:
    branches:
      - "**"  # Runs on all branches

This means:

Run the workflow whenever code is pushed to a branch.

The:

"**"

pattern matches essentially all branch names.

For example:

main
develop
feature/ansible-lint
feature/TICKET-description
bugfix/test

A push to any of these branches can trigger the workflow.

Example

You run:

git add .
git commit -m "Update Ansible playbook"
git push

GitHub detects the push and starts this workflow.

4. Jobs
jobs:
  automation:

A GitHub Actions workflow can contain one or more jobs.

Here there is one job:

automation

You can think of the job as:

"Everything I want GitHub Actions to execute for this pipeline."

5. Which machine runs the job?
runs-on: ["linux", "self-hosted", "ubuntu", "custom-runner", "docker"]

This tells GitHub:

Find a self-hosted runner that has all of these labels.

The runner needs labels matching:

linux
self-hosted
ubuntu
custom-runner
docker

So this is not necessarily a GitHub-hosted Ubuntu runner.

It is looking for your own self-hosted machine configured as a GitHub Actions runner.

For example:

Your Ubuntu Server
       │
       ├── GitHub Actions Runner
       ├── Docker
       ├── Linux
       └── custom-runner label

This is useful in enterprise environments where you want CI/CD jobs to execute on infrastructure you control.

6. Steps
steps:

This begins the individual actions that the job will perform.

The workflow executes these steps in order.

7. Checkout the repository
- name: Checkout Repository
  uses: actions/checkout@v4

This downloads your GitHub repository onto the runner.

Without this step, the runner doesn't have your code.

For example, suppose your repository contains:

ansible-project/
├── playbooks/
├── roles/
├── inventory/
├── Dockerfile
└── requirements.yml

The checkout action makes those files available on the runner.

8. Fetch complete Git history
with:
  fetch-depth: 0

Normally actions/checkout performs a shallow checkout.

For example:

Latest commit
     ↓
Repository

With:

fetch-depth: 0

GitHub retrieves the complete Git history.

That can be useful if your pipeline needs:

Git history
tags
commit information
versioning
comparisons between commits
9. Checkout the current Git reference
ref: ${{ github.ref }}

${{ }} is GitHub Actions' expression syntax.

${{ github.ref }}

represents the Git reference that triggered the workflow.

For example, if you push to:

feature/ansible-lint

GitHub knows that this branch triggered the workflow.

The checkout step therefore checks out the appropriate reference.

10. Configure AWS credentials
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2

This action configures AWS authentication for the runner.

Then:

with:

provides the configuration.

11. AWS Access Key
aws-access-key-id: ${{ secrets.STAGE_DEPLOY_USER_AWS_ACCESS_KEY_ID }}

The AWS access key is retrieved from GitHub's encrypted repository/environment secrets.

The important part is:

secrets.

This means the actual credential isn't stored directly in the YAML.

Instead, GitHub stores something like:

STAGE_DEPLOY_USER_AWS_ACCESS_KEY_ID

and the workflow references it.

12. AWS Secret Access Key
aws-secret-access-key: ${{ secrets.STAGE_DEPLOY_USER_AWS_SECRET_ACCESS_KEY }}

Same concept.

The secret AWS credential is stored in GitHub Secrets and injected into the workflow at runtime.

So you don't want this:

aws-secret-access-key: abc123...

You want the secret stored securely and referenced like:

${{ secrets.NAME }}
13. AWS Region
aws-region: ${{ vars.STAGE_AWS_DEFAULT_REGION }}

This tells the AWS action which AWS Region to use.

For example:

us-east-1

or:

us-east-2

Unlike secrets, this example uses:

vars.

which represents a GitHub Actions variable.

14. Login to Amazon ECR
- name: Login to Amazon ECR

This step authenticates Docker against Amazon Elastic Container Registry (ECR).

ECR is AWS's container image registry.

Think:

Docker Hub       → public/third-party registry
Amazon ECR       → AWS container registry
15. Get ECR authentication password
aws ecr get-login-password --region ${{ vars.STAGE_AWS_DEFAULT_REGION }}

AWS generates a temporary authentication password for ECR.

The region comes from:

${{ vars.STAGE_AWS_DEFAULT_REGION }}
16. Pipe the password into Docker
|
docker login --username AWS --password-stdin

The | means:

Take the output from the AWS command and pass it into the next command.

So:

aws ecr get-login-password
          ↓
      password
          ↓
docker login

This avoids putting the password directly on the command line.

17. ECR registry address
${{ vars.STAGE_AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.STAGE_AWS_DEFAULT_REGION }}.amazonaws.com

This constructs your AWS ECR registry URL.

For example, conceptually:

123456789012.dkr.ecr.us-east-1.amazonaws.com

Where:

123456789012 = AWS Account ID

us-east-1 = AWS Region

So the complete command is effectively:

docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
18. Build the Docker image
- name: Build Docker Image
  run: |
    docker build -t ansible-lint:latest .

This tells Docker:

Build an image using the Dockerfile in the current directory.

The:

.

means:

Use the current directory as the Docker build context.

And:

-t ansible-lint:latest

gives the image:

Repository: ansible-lint
Tag: latest

So you get:

ansible-lint:latest
19. Run Ansible Lint inside the container
- name: Run Ansible Lint inside the container
  run: |
    docker run --rm ansible-lint:latest ansible-lint

This is the most important part of the pipeline.

Docker starts a container from:

ansible-lint:latest

and executes:

ansible-lint

inside that container.

What is --rm?
--rm

means:

Automatically delete the container after it finishes.

Without --rm, you could accumulate stopped containers on your runner.

With it:

Start container
      ↓
Run ansible-lint
      ↓
Finish
      ↓
Container removed

The Docker image remains.

20. What does Ansible Lint do?

ansible-lint checks your Ansible code for problems and bad practices.

For example:

- name: Install nginx
  apt:
    name: nginx

Ansible Lint might identify issues involving:

YAML formatting
deprecated modules
incorrect module usage
missing best practices
idempotency problems
security practices
task naming
risky configurations

So your pipeline becomes a quality gate.

21. What happens if Ansible Lint fails?

Suppose your Ansible code contains a linting violation.

Then:

docker run
      ↓
ansible-lint
      ↓
ERROR
      ↓
Exit code != 0
      ↓
GitHub Actions job FAILS

The developer can see the failure in GitHub Actions.

If lint passes:

ansible-lint
     ↓
PASS
     ↓
GitHub Actions
     ↓
SUCCESS


One important issue with this workflow

There is something I would change in this particular pipeline.

You authenticate to ECR:

- name: Login to Amazon ECR

but you never push an image to ECR.

You're building:

ansible-lint:latest

and running it locally:

docker run --rm ansible-lint:latest ansible-lint

Therefore, ECR login isn't actually required for the current workflow.

The current flow is:

GitHub
   ↓
Self-hosted Runner
   ↓
Checkout
   ↓
AWS Login
   ↓
ECR Login       ← Not currently used
   ↓
Docker Build
   ↓
Ansible Lint

If your intention is simply:

Build a Docker container and use it to lint my Ansible code

you can remove the AWS credentials and ECR login entirely.

It would become:

GitHub
   ↓
Self-hosted Ubuntu Runner
   ↓
Checkout Repository
   ↓
Build Ansible-Lint Docker Image
   ↓
Run Ansible Lint
   ↓
PASS / FAIL

If your intention is instead:

Build the image, push it to ECR, and use that image later

then the ECR login makes sense, but you would need an additional:

docker tag ...
docker push ...

step.

One more security improvement

Your current workflow uses:

aws-access-key-id
aws-secret-access-key

For a modern GitHub Actions → AWS deployment pipeline, I would generally prefer GitHub OIDC + an AWS IAM role rather than long-lived AWS access keys.

The architecture would be:

GitHub Actions
      │
      │ OIDC token
      ▼
AWS IAM Role
      │
      ▼
Temporary AWS credentials
      │
      ▼
ECR / AWS resources

That eliminates the need to store a permanent AWS secret access key in GitHub.
