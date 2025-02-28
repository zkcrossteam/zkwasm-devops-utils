# zkwasm-devops-utils

## Guide to Converting a zkWASM APP Project into a DevOps K8s Project

This guide provides detailed steps to transform your zkWASM project into a DevOps Kubernetes project.

### Prerequisites
- Git installed
- Access to the zkWASM APP Project GitHub repository with Actions enabled
- Node.js and npm installed
- Kubernetes cluster access (for deployment)
- For other requirements, please refer to the [zkwasm development recipe](https://jupiterxiaoxiaoyu.github.io/zkwasm-development-recipe/getting-started/Setup%20Environment.html)

### Local Setup

**Note:** For steps 1-4 (adding configuration files, modifying Makefile, adding CI/CD workflow, and adding Dockerfile), please refer to the [Project Configuration Instructions](#project-configuration-instructions) section. You can simply copy all the necessary files from this repository to your zkWASM APP project.

1. **Add Required Configuration Files**
   - In your repository, add the following files (Already provided in the repo):
     - **Helm Chart Generation Script**
       - A script to generate Helm charts with these important parameters:
         - `CHART_NAME`: Set to your GitHub repo name (e.g., helloworld-rollup)
         - `ALLOWED_ORIGINS`: Configure CORS settings with comma-separated domain names
         - `DEPLOY_VALUE`: Set to `true` (default) to enable task submission
         - `REMOTE_VALUE`: Set to `true` (default) for remote synchronization
         - `AUTO_SUBMIT_VALUE`: Configure auto submission (optional)
         - `IMAGE_VALUE`: MD5 hash of your WASM file, will be automatically updated by the Makefile
         - Please leave the parameters as "" if you don't want to set them to "true"

     - **Environment Variables**
       - Add a `.env` file to configure essential environment variables
       ```bash
       # Create or edit the .env file
       nano .env
       
       # Example .env content:
       # SERVER_ADMIN_KEY=123
       ```

2. **Modify the Makefile**
   - Update your Makefile to include (Already provided in the repo):
     - A build section that generates WASM files, calculates MD5 hashes, and copies artifacts to the build-artifacts directory
     - Automatic updating of the `IMAGE_VALUE` in the Helm chart generation script
   ```bash
   # Edit the Makefile
   nano Makefile
   
   # Example build section:
   # build: ./src/admin.pubkey ./ts/src/service.js
   #   wasm-pack build --release --out-name application --out-dir pkg
   #   wasm-opt -Oz -o $(INSTALL_DIR)/application_bg.wasm pkg/application_bg.wasm
   #   cp pkg/application_bg.wasm $(INSTALL_DIR)/application_bg.wasm
   #   cd $(RUNNING_DIR) && npx tsc && cd -
   #   echo "MD5:"
   #   # Calculate MD5 and convert to uppercase
   #   $(eval MD5_VALUE := $(shell md5sum $(INSTALL_DIR)/application_bg.wasm | awk '{print $$1}' | tr 'a-z' 'A-Z'))
   #   echo "Calculated MD5: $(MD5_VALUE)"
   #   # Create build artifacts directory
   #   mkdir -p $(BUILD_ARTIFACTS_DIR)/application
   #   # Copy necessary WASM files to build artifacts directory
   #   cp $(INSTALL_DIR)/application_bg.wasm $(BUILD_ARTIFACTS_DIR)/application/
   #   cp $(INSTALL_DIR)/application_bg.wasm.d.ts $(BUILD_ARTIFACTS_DIR)/application/
   #   # Record MD5 to file
   #   echo "$(MD5_VALUE)" > $(BUILD_ARTIFACTS_DIR)/wasm.md5
   #   # Update IMAGE_VALUE in generate-helm.sh
   #   sed -i 's/^IMAGE_VALUE=.*$$/IMAGE_VALUE="$(MD5_VALUE)"/' scripts/generate-helm.sh
   ```

3. **Add CI/CD Workflow**
   - Add a GitHub Actions workflow file at `.github/workflows/ci-cd.yml` (Already provided in the repo):
     - Configure which branches and tags trigger the build process
     - Set up the build and deployment steps using pre-built WASM files
   ```bash
   # Create the workflows directory if it doesn't exist
   mkdir -p .github/workflows
   
   # Edit the CI/CD workflow file
   nano .github/workflows/ci-cd.yml
   ```

4. **Add Dockerfile for CI/CD**
   - Add a Dockerfile (Already provided in the repo) to build your project image as part of the CI/CD pipeline.
   ```bash
   # Edit the Dockerfile for CI/CD
   nano Dockerfile.ci
   ```

5. **Build TypeScript Components and WASM Files Locally**
   ```bash
   # Navigate to the TypeScript directory
   cd ts
   
   # Install dependencies
   npm install
   
   # Compile TypeScript
   npx tsc
   
   # Return to the project root
   cd ..
   
   # Build the WASM files, generate artifacts, and update Helm charts
   make build
   
   # Verify the generated Helm chart
   ls -la helm-charts/
   ```

6. **Test the Publish Script**
   - Test the publish script and fix any issues. Sometimes there might be errors when running without the `-n` flag.
   - If needed, modify `ts/publish.sh`.
   ```bash
   # Make the publish script executable
   chmod +x ts/publish.sh
   
   # Test the publish script
   cd ts && ./publish.sh -n && cd ..
   
   # If needed, edit the publish script
   nano ts/publish.sh
   ```

7. **Push to GitHub**
   - Ensure your GitHub repository has Actions enabled.
   ```bash
   # Add all files to git
   git add .
   
   # Commit changes
   git commit -m "Configure DevOps setup for zkWASM project"
   
   # Push to GitHub
   git push origin main
   ```

### Kubernetes Deployment

1. **Access Your Cluster**
   - Connect to your Kubernetes cluster using the appropriate credentials.
   ```bash
   # Example for GKE
   gcloud container clusters get-credentials CLUSTER_NAME --zone ZONE --project PROJECT_ID
   
   # Example for AWS EKS
   aws eks update-kubeconfig --name CLUSTER_NAME --region REGION
   
   # Verify connection
   kubectl cluster-info
   ```

2. **Clone the Repository**
   ```bash
   # Clone your repository
   git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
   
   # Navigate to the repository
   cd YOUR_REPO
   ```

3. **Create a Dedicated Namespace and Secrets**
   - Note: The namespace determines your service API URL, which will be in the format `https://rpc.<namespace>.zkwasm.ai`
   ```bash
   # Create a namespace for your project
   kubectl create namespace YOUR_NAMESPACE
   
   # Create Kubernetes secrets
   kubectl create secret generic app-secrets \
   --from-literal=SETTLER_PRIVATE_ACCOUNT='settler-key-for-the-namespace' \
   --from-literal=SERVER_ADMIN_KEY='admin-key-for-the-namespace' \
   --namespace=YOUR_NAMESPACE
   ```

4. **Deploy with Helm**
   ```bash
   # Install the Helm chart
   helm install YOUR_RELEASE_NAME ./helm-charts/YOUR_CHART_NAME -n YOUR_NAMESPACE
   
   # Example:
   # helm install holdit-release ./helm-charts/holdit-devops -n holdit
   ```

5. **Monitor Deployment**
   ```bash
   # Watch the pods being created
   kubectl get pods -n YOUR_NAMESPACE -w
   
   # Check pod logs if needed
   kubectl logs POD_NAME -n YOUR_NAMESPACE
   
   # Check deployment status
   kubectl get deployments -n YOUR_NAMESPACE
   ```

6. **Access the RPC Service**
   - Your service will be available at: `https://rpc.<namespace>.zkwasm.ai`
   ```bash
   # Check the ingress status
   kubectl get ingress -n YOUR_NAMESPACE
   
   # Test the service
   curl https://rpc.YOUR_NAMESPACE.zkwasm.ai/health
   ```

### Project Configuration Instructions

To adapt this setup for your zkWASM project:
- Modify the Helm script:
  ```bash
  # Edit the generate-helm.sh script
  nano scripts/generate-helm.sh
  
  # Update CHART_NAME to match your project name
  # Example: CHART_NAME="your-project-name"
  
  # Configure ALLOWED_ORIGINS for CORS settings
  # Example: ALLOWED_ORIGINS="https://example.com,https://app.example.com"
  
  # Set deployment options
  # DEPLOY_VALUE="true"
  # REMOTE_VALUE="true"
  # AUTO_SUBMIT_VALUE=""
  
  # The IMAGE_VALUE will be automatically updated by the Makefile
  # when you run 'make build'
  ```
- Update the `.env` file as needed for your project
  ```bash
  # Edit the .env file
  nano .env
  ```
- Adjust the `ci-cd.yml` workflow file to match your project's build requirements
  ```bash
  # Edit the CI/CD workflow file
  nano .github/workflows/ci-cd.yml
  ```
- Copy all configured files to the root directory of your zkWASM project
  ```bash
  # Example of copying files to another project
  cp -r scripts/ /path/to/your/zkwasm/project/
  cp -r .github/ /path/to/your/zkwasm/project/
  cp Dockerfile.ci /path/to/your/zkwasm/project/
  cp .env /path/to/your/zkwasm/project/
  ```

### Troubleshooting

- If GitHub Actions fail, check the workflow logs for specific errors
  ```bash
  # View GitHub Actions logs through the GitHub web interface
  # Navigate to your repository > Actions > Select the failed workflow
  ```
- For Kubernetes deployment issues, use `kubectl describe pod <pod-name> -n <namespace>` to get detailed error information
  ```bash
  # Get pod names
  kubectl get pods -n YOUR_NAMESPACE
  
  # Describe a specific pod
  kubectl describe pod POD_NAME -n YOUR_NAMESPACE
  
  # Check pod logs
  kubectl logs POD_NAME -n YOUR_NAMESPACE
  ```
- Ensure all required secrets are properly configured in your Kubernetes namespace
  ```bash
  # List Kubernetes secrets in your namespace
  kubectl get secrets -n YOUR_NAMESPACE
  ```

  