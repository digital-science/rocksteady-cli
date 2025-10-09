# RockSteady CLI

A command-line interface for building and deploying Docker images to AWS ECR and notifying the RockSteady deployment service. This tool is designed to be used in CI/CD environments.

## Features

* Builds Docker images from a local `Dockerfile`.
* Tags images with git branch, commit SHA, and build number for easy tracking.
* Pushes images to a specified AWS ECR repository.
* Notifies a RockSteady server of a new successful build, triggering deployments.
* Leverages Docker layer caching to speed up builds by reusing layers from previous images on the same branch.

## How it works

The `rocksteady` script has two main subcommands: `build` and `deploy`.

### Build and Push an Image

This command builds the Docker image, tags it, and pushes it to your ECR repository.

```bash
rocksteady build
```

This command will:

1. Authenticate with AWS ECR using the provided credentials.
2. Attempt to pull the latest image for the current branch to use as a build cache (`--cache-from`). This makes builds faster by reusing unchanged layers.
3. Build the Docker image from the `Dockerfile` in the current directory.
4. Tag the image with multiple tags:
    * `:build-<build_num>`
    * `:<branch>-<build_num>`
    * `:<branch>-latest`
    * `:<commit_sha>`
    * `:latest` (if the branch is `main` or `master`)
5. Push all generated tags to the ECR repository.

### Notify RockSteady of a New Build

This command sends a webhook to the RockSteady server to notify it that a new build is available for deployment.

```bash
rocksteady deploy ROCKSTEADY_URL
```

* `ROCKSTEADY_URL`: The URL of the RockSteady server's webhook endpoint. This can also be provided via the `ROCKSTEADY_SERVER` environment variable.

## Configuration

The `rocksteady-cli` is configured entirely through environment variables. The script checks for multiple environment variables for the same configuration. Right now it's being invoked and configured from Altmetric's CircleCI account.

### Required Environment Variables

| Variable                      | Fallback Variable             | Description                                                              |
| ----------------------------- | ----------------------------- | ------------------------------------------------------------------------ |
| `ROCKSTEADY_PROJECT`          | `CIRCLE_PROJECT_REPONAME`     | The name of the project in RockSteady.                                   |
| `ECR_BASE`                    |                               | The base URI of your ECR repository (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com`). |
| `ECR_AWS_ACCESS_KEY_ID`       | `AWS_ACCESS_KEY_ID`           | The AWS access key ID for ECR authentication.                            |
| `ECR_AWS_SECRET_ACCESS_KEY`   | `AWS_SECRET_ACCESS_KEY`       | The AWS secret access key for ECR authentication.                        |
| `ECR_AWS_REGION`              |                               | The AWS region where your ECR repository is located.                     |
| `CIRCLE_BUILD_NUM`            |                               | The build number, used for tagging.                                      |
| `CIRCLE_BRANCH`               |                               | The git branch name, used for tagging.                                   |
| `CIRCLE_SHA1`                 |                               | The git commit SHA, used for tagging.                                    |

### Optional Environment Variables

| Variable                      | Fallback Variable             | Description                                                              |
| ----------------------------- | ----------------------------- | ------------------------------------------------------------------------ |
| `ECR_REPO`                    | `ROCKSTEADY_PROJECT`          | The name of the ECR repository. Defaults to the project name.            |
| `ROCKSTEADY_SERVER`           |                               | The URL for the RockSteady server. Can be overridden by the command line argument in `deploy`. |
| `CF_ACCESSC_ID`               |                               | Cloudflare Access Client ID for authenticating with the RockSteady webhook. |
| `CF_ACCESS_SECRET`            |                               | Cloudflare Access Client Secret for authenticating with the RockSteady webhook. |
| `SIDEKIQ_PRO_TOKEN`           |                               | Build-time variable passed to `docker build`.                            |
| `DSUI_GITHUB_TOKEN`           |                               | Build-time variable passed to `docker build`.                            |

## Usage

Add the `rocksteady` script to your CI/CD pipeline. Here’s an example of how to use it in a CircleCI configuration:

```yaml
deploy:
  docker:
    - image: altmetric/ci:rocksteady-deploy-latest

      auth:
        username: $DOCKERHUB_USERNAME
        password: $DOCKERHUB_PASSWORD

  steps:
    - setup_remote_docker:
        version: default

    - checkout

    - run:
        command: deploy
```
