# cloudbuild-for-CI-argocd-for-CD

you create a CI/CD pipeline that automatically builds a container image from committed code, stores the image in Artifact Registry, updates a Kubernetes manifest in a Git repository, and deploys the application to Google Kubernetes Engine using that manifest using ARGO CD.

![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/143b3bdc-3c95-43f2-b5b8-535d0bead510)

# For this lab you will create 2 Git repositories:

app repository: contains the source code of the application itself
env repository: contains the manifests for the Kubernetes Deployment

# Workflow
1. When you push a change to the app repository,
2. the Cloud Build pipeline runs tests, builds a container image, and pushes it to Artifact Registry.
3. After pushing the image, Cloud Build updates the Deployment manifest and pushes it to the env repository.
4. This triggers another Cloud Build pipeline that applies the manifest to the GKE cluster and, if successful, stores the manifest in another branch of the env repository.

# Description
1. The app and env repositories are kept separate because they have different lifecycles and uses.
2. The main users of the app repository are actual humans and this repository is dedicated to a specific application.
3. The main users of the env repository are automated systems (such as Cloud Build), and this repository might be shared by several applications.
4. The env repository can have several branches that each map to a specific environment (you only use production in this lab) and reference a specific container image, whereas the app repository does not.

![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/00e03bf7-6468-424a-8cfc-69d10e59e549)

# When you finish this lab, you have a system where you can easily:
1. Distinguish between failed and successful deployments by looking at the Cloud Build history.
2. Access the manifest currently used by looking at the production branch of the env repository.
3. Rollback to any previous version by re-executing the corresponding Cloud Build build.

# Resource creating

1. Create Kubernetes Engine clusters
2. Create Cloud Source Repositories
3. Trigger Cloud Build from Cloud Source Repositories
4. Automate tests and publish a deployable container image via Cloud Build
5. Manage resources deployed in a Kubernetes Engine cluster via Cloud Build


# Task 1: Setting up the Environment 

1. Open Cloud Shell
   ```
   export PROJECT_ID=$(gcloud config get-value project)
   export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
   export REGION=us-west1
   gcloud config set compute/region $REGION
   ```
2. Run the following command to enable the APIs for GKE, Cloud Build, Cloud Source Repositories and Container Analysis:
   ```
   gcloud services enable container.googleapis.com \
   cloudbuild.googleapis.com \
   sourcerepo.googleapis.com \
   containeranalysis.googleapis.com
   ```
3. Create an Artifact Registry Docker repository in us-west1 region
   ```
   gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=$REGION
   ```
4. Create a GKE cluster to deploy the sample application of this lab:
   ```
   gcloud container clusters create hello-cloudbuild --num-nodes 1 --region $REGION
   ``` 
5. Congirue GIT
   ```
   git config --global user.email "you@example.com"
   git config --global user.name "Your Name"
   ```

# Task 2. Create the Git repositories in Cloud Source Repositories

1. Create two repositories named hello-cloudbuild-app and hello-cloudbuild-env
   ```
   gcloud source repos create hello-cloudbuild-app
   gcloud source repos create hello-cloudbuild-env
   cd ~
   git clone https://github.com/GoogleCloudPlatform/gke-gitops-tutorial-cloudbuild hello-cloudbuild-app
   cd ~/hello-cloudbuild-app
   PROJECT_ID=$(gcloud config get-value project)
   git remote add google "https://source.developers.google.com/p/${PROJECT_ID}/r/hello-cloudbuild-app"
   ```
2. Manually create docker image using cloud build
   ```
   cd ~/hello-cloudbuild-app
   COMMIT_ID="$(git rev-parse --short=7 HEAD)"
   gcloud builds submit --tag="${REGION}-docker.pkg.dev/${PROJECT_ID}/my-repository/hello-cloudbuild:${COMMIT_ID}" .
   ```
3. After the build finishes, go to Artifact Registry > verify that your new container image is available in Click my-repository.

![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/b812efc5-8a67-46b4-86dd-98e8a40c82b0)

# Task 3. Create the Continuous Integration (CI) pipeline

1. configure Cloud Build to automatically run a small unit test,
  * build the container image, and then push it to Artifact Registry.
  * Pushing a new commit to Cloud Source Repositories triggers this pipeline automatically.
  * The cloudbuild.yaml file already included in the code is the pipeline's configuration.

![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/34548b66-2054-473e-b261-1ac92534212c)

![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/6784d8c2-8bcc-4049-8df6-3222ba31e159)

2. To start this trigger, run the following command:
   ```
   cd ~/hello-cloudbuild-app
   git push google master
   ```
   * In the Cloud console, go to Cloud Build > Dashboard > and verify the cloud build is successfull

# Task 5. Create the Test Environment and CD pipeline
1. Cloud Build is also used for the continuous delivery pipeline.
2. The pipeline runs each time a commit is pushed to the candidate branch of the hello-cloudbuild-env repository.
3. The pipeline applies the new version of the manifest to the Kubernetes cluster and, if successful, copies the manifest over to the production branch.

**This process has the following properties:**
a. The candidate branch is a history of the deployment attempts.
b.The production branch is a history of the successful deployments.
c. You have a view of successful and failed deployments in Cloud Build.
d. You can rollback to any previous deployment by re-executing the corresponding build in Cloud Build.
e. A rollback also updates the production branch to truthfully reflect the history of deployments.

**Grant Cloud Build access to GKE**

1. application in your Kubernetes cluster, Cloud Build needs the Kubernetes Engine Developer Identity and Access Management role.
   ```
   PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"
   gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com --role=roles/container.developer
   ```
**You need to initialize the hello-cloudbuild-env repository with two branches (production and candidate)**
**Cloud Build configuration file describing the deployment process. The first step is to clone the hello-cloudbuild-env repository and create the production branch. It is still empty.**

2. Initiate two branches
   ```
   cd ~
   gcloud source repos clone hello-cloudbuild-env
   cd ~/hello-cloudbuild-env
   git checkout -b production
   cd ~/hello-cloudbuild-env
   cp ~/hello-cloudbuild-app/cloudbuild-delivery.yaml ~/hello-cloudbuild-env/cloudbuild.yaml
   git add .
   git commit -m "Create cloudbuild.yaml for deployment"
   git checkout -b candidate
   git push origin production
   git push origin candidate
   ```
3. Grant the Source Repository Writer IAM role to the Cloud Build service account for the hello-cloudbuild-env repository:
   ```
   PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} \
   --format='get(projectNumber)')"
   cat >/tmp/hello-cloudbuild-env-policy.yaml <<EOF
   bindings:
   - members:
     - serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com
     role: roles/source.writer
   EOF
   ```
   ```
   gcloud source repos set-iam-policy \
   hello-cloudbuild-env /tmp/hello-cloudbuild-env-policy.yaml
   ```

# Create the trigger for the continuous delivery pipeline
1. In the Cloud console, go to Cloud Build > Triggers.
2. Click Create Trigger.
3. In the Name field, type hello-cloudbuild-deploy.
4. Under Event, select Push to a branch.
5. Under Source, select hello-cloudbuild-env as your Repository and ^candidate$ as your Branch.
6. Under Build configuration, select Cloud Build configuration file.
7. In the Cloud Build configuration file location field, type cloudbuild.yaml after the /.
8. Click Create.

# Copy the extended version of the cloudbuild.yaml file for the app repository and push to repository:
```
cd ~/hello-cloudbuild-app
cp cloudbuild-trigger-cd.yaml cloudbuild.yaml
cd ~/hello-cloudbuild-app
git add cloudbuild.yaml
git commit -m "Trigger CD pipeline"
git push google master
```

# Task 6. Review Cloud Build Pipeline
![image](https://github.com/Pruthvi2340/cloudbuild-for-CI-argocd-for-CD/assets/152501425/0b23a8cb-3624-4c03-8c90-8981f320e22a)

# Task 7. Test the complete pipeline
1. go to Kubernetes Engine > Services & Ingress and access with public ip in the browser.
2. In Cloud Shell, replace "Hello World" with "Hello Cloud Build", both in the application and in the unit test:
   ```
   cd ~/hello-cloudbuild-app
   sed -i 's/Hello World/Hello Cloud Build/g' app.py
   sed -i 's/Hello World/Hello Cloud Build/g' test_app.py
   ```
3. Commit and push the change to Cloud Source Repositories:
   ```
   git add app.py test_app.py
   git commit -m "Hello Cloud Build"
   git push google master
   ```

# Task 8. Test the rollback

1. In the Cloud console, go to Cloud Build > Dashboard.
2. Click on View all link under Build History for the hello-cloudbuild-env repository.
3. Click on the second most recent build available.
4. Click Rebuild.
