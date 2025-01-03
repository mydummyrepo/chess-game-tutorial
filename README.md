# React + Vite

This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react/README.md) uses [Babel](https://babeljs.io/) for Fast Refresh
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react-swc) uses [SWC](https://swc.rs/) for Fast Refresh



---

Deploying Docker images as applications to a Google Kubernetes Engine (GKE) cluster using a GitHub Actions pipeline involves the following steps:

---

### **1. Prerequisites**
- **GKE Cluster:** A running GKE cluster with proper access setup.
- **Docker Image:** A Dockerfile or pre-built Docker image.
- **Google Cloud SDK:** Installed locally or configured in GitHub Actions using the `google-github-actions` workflow.
- **Service Account:** A Google Cloud service account with appropriate permissions for GKE and Container Registry/Artifact Registry.
- **Kubernetes Manifest Files:** Deployment YAML files for Kubernetes.

---

### **2. Steps to Set Up the GitHub Actions Pipeline**

#### **Step 1: Configure Google Cloud Credentials**
1. Create a **Service Account**:
   - Go to the **Google Cloud Console** > IAM & Admin > Service Accounts.
   - Create a service account with roles:
     - **Kubernetes Engine Developer**
     - **Storage Admin** (for pushing Docker images to the registry)
   - Generate a JSON key for the service account.

2. Add the JSON key to **GitHub Secrets**:
   - Go to your GitHub repository > Settings > Secrets and variables > Actions > Add secret.
   - Add the JSON key as a secret (e.g., `GCLOUD_AUTH`).

---

#### **Step 2: Write the GitHub Actions Workflow**
Create a `.github/workflows/deploy-to-gke.yml` file:

```yaml
name: Deploy to GKE

on:
  push:
    branches:
      - main  # Trigger on pushes to the main branch

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    # Step 1: Checkout code
    - name: Checkout repository
      uses: actions/checkout@v3

    # Step 2: Set up Google Cloud SDK
    - name: Set up Google Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        version: 'latest'
        service_account_key: ${{ secrets.GCLOUD_AUTH }}
        project_id: your-gcp-project-id

    # Step 3: Authenticate Docker to push images to Google Container/Artifact Registry
    - name: Authenticate Docker with Google Container Registry
      run: gcloud auth configure-docker

    # Step 4: Build and push Docker image
    - name: Build and Push Docker Image
      run: |
        IMAGE_NAME=gcr.io/your-gcp-project-id/your-app-name
        docker build -t $IMAGE_NAME:$GITHUB_SHA .
        docker push $IMAGE_NAME:$GITHUB_SHA

    # Step 5: Set up kubectl
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'

    # Step 6: Authenticate kubectl to the GKE cluster
    - name: Authenticate kubectl
      run: |
        gcloud container clusters get-credentials your-cluster-name \
          --region your-cluster-region \
          --project your-gcp-project-id

    # Step 7: Deploy to GKE
    - name: Deploy to GKE
      run: |
        kubectl set image deployment/your-deployment-name your-container-name=gcr.io/your-gcp-project-id/your-app-name:$GITHUB_SHA
        kubectl rollout status deployment/your-deployment-name
```

---

### **3. Key Configuration Details**
- **Docker Image:**
  - Replace `your-app-name` with your application's name.
  - Use `$GITHUB_SHA` as the image tag for traceability.

- **Kubernetes Deployment:**
  - Ensure your Kubernetes deployment YAML specifies the correct container name.
  - Use `kubectl set image` to update the container image in the deployment.

- **GKE Cluster:**
  - Replace `your-cluster-name`, `your-cluster-region`, and `your-gcp-project-id` with your actual GKE details.

---

### **4. Deployment YAML File Example**
Make sure your Kubernetes deployment YAML file is set up properly:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-deployment-name
  labels:
    app: your-app-name
spec:
  replicas: 2
  selector:
    matchLabels:
      app: your-app-name
  template:
    metadata:
      labels:
        app: your-app-name
    spec:
      containers:
      - name: your-container-name
        image: gcr.io/your-gcp-project-id/your-app-name:latest
        ports:
        - containerPort: 80
```

---

### **5. Additional Tips**
1. **Rollback Support:** Add a GitHub Actions step to rollback in case of failed deployments:
   ```yaml
   - name: Rollback on failure
     if: failure()
     run: |
       kubectl rollout undo deployment/your-deployment-name
   ```

2. **Testing in Staging:** Deploy to a staging environment before production by adding environment-specific steps.

3. **Use Artifact Registry (Optional):** Switch from Container Registry to Artifact Registry for better features:
   - Replace `gcr.io` with `LOCATION-docker.pkg.dev` in the image URL.

---

### **6. Workflow Summary**
- **Push Trigger:** The pipeline is triggered on a push to the `main` branch.
- **Docker Build & Push:** Builds and pushes the Docker image to GCR.
- **Deploy to GKE:** Updates the Kubernetes deployment with the new image.

With this setup, every time you push changes to the `main` branch, your application will automatically build, push, and deploy to your GKE cluster.