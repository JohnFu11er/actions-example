# actions-example

### Explanation of the Workflow Steps:
- Trigger the Workflow:
  - The workflow triggers on push events to the main branch and tags like v* (e.g., v1.0, v2.1).
  - You can customize the branches and tag patterns according to your needs.
- Checkout Code:
  - This step checks out your repository’s code, allowing the workflow to access the source code and Dockerfile.

- Set up Docker Buildx (optional but recommended for multi-platform builds):
  - This allows building Docker images on multiple platforms (like linux/arm64 or linux/amd64).

- Log in to Docker Hub (or GHCR):
  - The action logs in to Docker Hub using your Docker credentials stored as GitHub secrets (DOCKER_USERNAME and DOCKER_PASSWORD).
  - If you're using GitHub Container Registry (GHCR), you can use ghcr.io as the registry in the docker push commands instead of Docker Hub.

- Build and Tag the Docker Image:
  - The image is tagged with the short commit hash ($COMMIT_SHA) and, if available, the tag from GitHub ($TAG). If the commit isn’t tagged, the image is tagged with latest.
  - docker build -t $IMAGE_NAME:$COMMIT_SHA -t $IMAGE_NAME:$TAG . will build the Docker image with both the commit hash and the tag as tags.

- Push the Docker Image:
  - The image is pushed to the specified registry with both the commit hash and tag.

### Key Points to Customize:
- Docker Registry: Replace mydockerhubuser/my-image with your Docker Hub or GHCR repository name.
- GitHub Secrets: Store your Docker Hub username and password as GitHub secrets (DOCKER_USERNAME, DOCKER_PASSWORD). If using GHCR, you may need to authenticate using a personal access token.
- Tags: The image will be tagged with both the commit hash ($COMMIT_SHA) and the tag ($TAG). If there is no tag, the image will be tagged as latest.

### Example of Git Tags:
- When you create a tag in Git, like v1.0, the workflow will trigger and use v1.0 as the tag for the Docker image.
- Resulting Tags:
  - Commit-based tag: mydockerhubuser/my-image:<commit_sha>
  - Tag-based tag: mydockerhubuser/my-image:v1.0
- This allows you to reference your Docker images by both the commit hash and version tag, which can be useful for reproducibility and versioning.
