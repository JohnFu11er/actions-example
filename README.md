# Actions-example
### Notes on setup
- Ensure that you grant `Read and write permissions` to your `GIHUB_TOKEN`
  - Navigate to:
    - Your repository > Settings > Actions > General > Workflow permissions
  - Select `Read and write permissions`
  - Click `Save`

### Explanation of the Workflow Steps:
- Trigger the Workflow:
  - The workflow triggers on
    - `pull_requests`:
      - To the main branch
      - Code:
        ```yaml
        # Trigger on pull request to the `main` branch
        pull_request:
          branches:
            - main
        ```
    - `push`
      - On any branch (feature or main)
      - Any kind of TAG that has been assigned to the repository
      - Code
        ```yaml
        # Trigger on push to any branch
        push:
          branches:
            - '**'  # Wildcard to match any branch
          tags:
            - '*'  # Wildcard to match any tag
        ```
      - You can customize the branches and tag patterns according to your needs.
        - Example:
          ```yaml
          tags:
            - 'v*'  # Wildcard that matches any tag starting with a 'v'
- Declare and initialize variables
  - This section creates variables that are accessible accross all steps in the workflow
    ```yaml
    # Declare and initialized variables
    env:
      TAG: ''
      IMAGE_NAME: ''
      COMMIT_SHA: ''
      IMAGE_REGITRY: ghrc.io                  # GitHub Container Registry address
      PLATFORMS: 'linux/amd64,linux/arm64'    # Indicates that the images will be built for amd64, and arm64 architectures
    ```
- Checkout Code:
  - This step checks out your repository’s code, allowing the workflow to access the source code and Dockerfile.
    ```yaml
    # Check out the repository
    - name: Checkout code
      uses: actions/checkout@v2
    ```
- Set up Docker Buildx (optional but recommended for multi-platform builds):
  - This allows building Docker images on multiple platforms (like linux/arm64 or linux/amd64).
    ```yaml
    # Set up Docker Buildx (optional but recommended for multi-platform builds)
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    ```

- Log in to Docker Hub (or GHCR):
  - The action logs in to  GitHub Container Registry (GHCR), using ghcr.io as the registry in the docker push commands instead of Docker Hub.
    ```yaml
    # Log in to GitHub Container Registry
    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v2
      with:
        registry: ghcr.io
        username: ${{ github.actor }}           # This is built in, and you do not need to set this variable
        password: ${{ secrets.GITHUB_TOKEN }}   # This is dynamically retreived from GitHub, and you do not need to set this variable
    ```
- Set the variables to be used in later steps
  - This step will set the value of the env variables based on the conditions of the commit, merge request, or tag
    ```yaml
    # Set the variables to be used in later steps
    - name: Set varialbes
      run: |
        # Assign variables
        IMAGE_REGISTRY="${{ env.IMAGE_REGITRY }}"
        REPO_NAME="${{ github.repository }}"
        REPO_NAME_LOWERCASE=$(echo "$REPO_NAME" | tr '[:upper:]' '[:lower:]')
        IMAGE_NAME="$IMAGE_REGISTRY/$REPO_NAME_LOWERCASE/my-image"
        COMMIT_SHA=$(git rev-parse --short HEAD)

        # If the commit is tagged, use the tag as the version,
        # unless there is no tag, then use the "latest" if on main branch.
        # Otherwise do not set an additional tag. All builds will have the
        # COMMIT_SHA used to tag the image
        if [[ $GITHUB_REF == refs/tags/* ]]; then
          TAG=$(echo $GITHUB_REF | sed 's/refs\/tags\///')
        elif [[ $GITHUB_REF == refs/heads/main ]]; then
          TAG="latest"
        else
          TAG=''
        fi

        # Persist the variables to the next stage in the workflow
        echo "IMAGE_NAME=$IMAGE_NAME" >> $GITHUB_ENV
        echo "COMMIT_SHA=$COMMIT_SHA" >> $GITHUB_ENV
        echo "TAG=$TAG" >> $GITHUB_ENV
    ```
- Build and push Docker image if there is a TAG or the branch is Main:
  - The image is tagged with the short commit hash ($COMMIT_SHA) and, if available, the tag from GitHub ($TAG). If the commit isn’t tagged but is on the `Main` branch, the image is tagged with `latest`.
  - A version of the image is built for each system architecture listed in the `platforms` section
    ```yaml
    # Build and push
    - name: Build and push Docker image if there is a TAG or the branch is Main
      if: env.TAG != '' || github.ref == 'refs/heads/main'
      uses: docker/build-push-action@v3
      with:
        context: .
        push: true
        tags: |
          "ghcr.io/${{ env.IMAGE_NAME }}:${{ env.TAG }}"
          "ghcr.io/${{ env.IMAGE_NAME }}:${{ env.COMMIT_SHA }}"
        platforms: ${{ env.PLATFORMS }}
      env:
        DOCKER_BUILDKIT: 1
    ```

- Build and push Docker image if there is no TAG and branch is not Main:
  - This is to be used for feature branch commits where you only use the `$COMMIT_SHA` for tagging the container image
  - A version of the image is built for each system architecture listed in the `platforms` section
    ```yaml
    # Build and push
    - name: Build and push Docker image if there is no TAG and branch is not Main
      if: env.TAG == '' && github.ref != 'refs/heads/main'
      uses: docker/build-push-action@v3
      with:
        context: .
        push: true
        tags: "ghcr.io/${{ env.IMAGE_NAME }}:${{ env.COMMIT_SHA }}"
        platforms: ${{ env.PLATFORMS }}
      env:
        DOCKER_BUILDKIT: 1
    ```
