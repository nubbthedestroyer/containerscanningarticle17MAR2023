# Container Scanning with Third Party Tools in a Cloud Environment
##### Michael Lucas

Hey Peraton Family!  My name is Michael Lucas and I am one of the latest additions to Peraton's Senior Cloud Team, working alongside Danny, Palani, Popsi, and our fearless leader, Brittain.  My experience over the last decade has been focused on cloud technologies and specifically container based DevOps.  In this way, I am definately a Kubernetes (and containers in general) fanboy and I figured I would share a little bit of that love with y'all.

If you're using containers in your DevOps pipeline, it's important to ensure that they're secure before they're deployed. Container scanning is a key part of this process, allowing you to identify vulnerabilities and other security issues in your container images before they're deployed to production.

Container scanning works by analyzing the contents of a container image and comparing them to known security vulnerabilities and best practices. This allows you to identify potential issues and take steps to address them before they can be exploited by attackers.

In this article, we'll explore the importance of container scanning in DevOps, and show you how to integrate container scanning into your own pipeline using popular tools like Aqua Security and Amazon ECR. 

I didn't want to make this an enormous walk through, so I'm just going to share a few key takeaways to consider and allow you to validate these opinions with you own experience.

Consider the bash script below.  This functionally encapsulates all the logic we need to add container scanning to really any CI/CD pipeline, but of course there are other more elegant ways to do it.  Here's a breakdown of what is going on in this script:

- Building the Docker container
    - In this step we simply seek to run a docker build against the Dockerfile in a given codebase.  If this script is being included in an existing pipeline where the container is built earlier, you don't need to build in this step.  In fact, you shouldn't.  The container scan absolutely needs to run against the actual image that is going to be deployed, not a rebuilt one (even if its the same code).
- Running AquaScanner against the image and grabbing the results.
    - AquaScanner is kind enough to offer JSON output, so lets parse it.
- Identify Critical Findings
    - If there are critical findings, we probably don't want to allow this container to be deployed into production, so in this case we are going to fail the script which will terminate the pipeline and alert our developers.
    - It's possible to explore some additional possibilities here, like maintaining a list of excluded or authorized vulnerabilities that don't cause the pipeline to fail, but this is enough for the purposes of this article.

```bash
#!/bin/bash

# Input variables
IMAGE_NAME=my-app
IMAGE_TAG=v1.0.0
REGION=us-east-1
ACCOUNT_ID=123456789012
REPOSITORY_NAME=my-repo

set -e

# Build the Docker image.
docker build -t $IMAGE_NAME:$IMAGE_TAG .

# Scan the Docker image with AquaScanner.
SCAN_OUTPUT=$(docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  registry.aquasec.com/scanner:5.0 \
  --image $IMAGE_NAME:$IMAGE_TAG \
  --format json \
  --silent)

# Check if there are any critical findings.
CRITICAL_FINDINGS=$(echo $SCAN_OUTPUT | jq '.[].findings[].severity' | grep CRITICAL)

if [ -z "$CRITICAL_FINDINGS" ]; then
  echo "No critical findings. Pushing Docker image to ECR..."
  # Push the Docker image to ECR.
  $(aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com)
  docker tag $IMAGE_NAME:$IMAGE_TAG $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPOSITORY_NAME:$IMAGE_TAG
  docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPOSITORY_NAME:$IMAGE_TAG
else
  echo "Critical findings detected. Exiting..."
  exit 1
fi
```

This is just one approach to a simple problem but there are so many other concepts in the container world worth exploring.  I would suggest having a look at this other articles to get your feet wet on the container scanning topic:

- "Container Security: The Importance of Vulnerability Scanning" - Aqua Security blog: https://blog.aquasec.com/container-security-importance-vulnerability-scanning
- "Getting Started with Container Scanning" - AWS Compute - Blog: https://aws.amazon.com/blogs/compute/getting-started-with-container-scanning/
- "Container Scanning with Trivy" - Sysdig blog: https://sysdig.com/blog/container-scanning-with-trivy/
- "Why you Need to Implement Continuous Container Security" - Twistlock blog: https://www.twistlock.com/2018/06/05/why-you-need-to-implement-continuous-container-security/
- "Container Security Best Practices: How to Build and Secure Containers" - Qualys blog: https://blog.qualys.com/container-security/2020/01/06/container-security-best-practices-how-to-build-and-secure-containers

Well thank you for exploring this with me, team!  I'm looking forward to all the great work we are going to do together.  If you have any questions about container scanning or containers in general, please feel free to reach out to me through any of the standard channels.
