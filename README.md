# Setup GKE Google
As a couple of our learning outcomes include things like, 'cloud native' & 'scalable architectures'. I want to deploy a simple Python application to a cloud hosted Kubernetes instance. I first checked if this was possible with Azure, but when I created an AKS, my quota got exceeded. So, my next solution is to take a look at Google's cloud platform and try to use the €280 of free credits.

I found a nice article that lists all the available free trials where I can manage a Kubernetes cluster on this link: https://github.com/learnk8s/free-kubernetes.

What I will learn by doing this:
- Using Google's Cloud platform
	- Making use of the 'Artifact Registry'
	- Making use of the 'Kubernetes Cluster'
- Kubernetes

Prerequisites:
- Google Cloud account
- Newly created project inside of Google Cloud (I named mine 'Kubernetes')
- Newly created GitHub repository
- Python Flask application (see code below)

## Python Project
For this demo, I have created a Python Flask application that has a single endpoint on the '/' which returns a 'Hello World!'.

Python Code:
```python
from flask import Flask  
  
app = Flask(__name__)  
  
  
@app.route('/')  
def hello_world():  
    return 'Hello, World!'  
  
  
if __name__ == '__main__':  
    app.run(host="0.0.0.0", port=8080)
```

Inside this project, there is also a simple Dockerfile, that looks as followed:
```Dockerfile
# Use the official Python base image  
FROM python:3.12-slim  
  
# Set the working directory inside the container  
WORKDIR /app  
  
# Copy the requirements file to the working directory  
COPY requirements.txt .  
  
# Install the Python dependencies  
RUN pip install --no-cache-dir -r requirements.txt  
  
# Copy the application code to the working directory  
COPY . .  
  
# Expose the port on which the application will run  
EXPOSE 8080  
  
# Run the FastAPI application using uvicorn server  
CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0", "--port=8080"]
```

## Artifact Registry
I will look into how to use GitHub actions to automatically build docker images and push those images to Google's artifact registry service. These images can be used to deploy to services like the Google Kubernetes engine.

First, I will set up an 'Artifact Registry' repository. Inside the newly created project, search in the top bar for 'Artifact Registry'. Once selected, it will send us to a page where we can enable the 'Artifact Registry API'. After the API is enabled, we can create a new repository and give it a name like 'kubernetes-demo' and select the region that is closest to you. Leave the rest of the options as the default value and create the repository.

### Workflow
Inside the created Python project, create a new directory `.github/workflows` and add the file `build.yml`.

Inside this file, we have to specify some steps to achieve building and pushing the Docker image into the 'Artifact Registry'.

The workflow triggers whenever there is a push to the `main` branch. It runs on an `ubuntu-latest` runner and begins by checking out the repository's code using `actions/checkout@v2`, to ensure the latest code is available.

Then it authenticates with Google Cloud using `google-github-actions/auth@v2`, configuring it with the Google Cloud project ID and service account key from the repository's secrets.

Next, it sets up the Google Cloud CLI (`gcloud`) using `google-github-actions/setup-gcloud@v2`.

The workflow proceeds to build a Docker image for the Flask application, tagging it as `europe-west4-docker.pkg.dev/secret-willow-423606-j1/kubernetes-demo/python:3.12-slim`, and pushes this image to the specified Google Container Registry.

The completed file should look like this:
```yml
name: Deploy to Flask to GKE  
  
on:  
  push:  
    branches:  
      - main  
  
jobs:  
  deploy:  
    runs-on: ubuntu-latest  
    steps:  
      - name: Checkout Code  
        uses: actions/checkout@v2  
  
      - name: Authenticate with Google Cloud  
        uses: 'google-github-actions/auth@v2'  
        with:  
          credentials_json: ${{ secrets.GOOGLE_APPLICATION_CREDENTIALS }}  
  
      - name: 'Set up Cloud SDK'  
        uses: 'google-github-actions/setup-gcloud@v2'  
  
      - name: Build and Push Docker Image  
        env:  
          GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}  
        run: |  
          gcloud auth configure-docker europe-west4-docker.pkg.dev  
          docker build -t europe-west4-docker.pkg.dev/$GOOGLE_PROJECT/kubernetes-demo/python:latest .  
          docker push europe-west4-docker.pkg.dev/$GOOGLE_PROJECT/kubernetes-demo/python:latest
```

After creating this file, we can commit the changes made to the code into the newly set up GitHub repo. But before pushing the actual changes, we need to add the credentials used in the workflow to our repository.

Commit changes by running:
```terminal
git add .
git commit -m"Added small python project, with build.yml to push to Google cloud Artifact Registry"
```

### Credentials
To set credentials, we need to be on the GitHub project page of where we need to deploy our application from. On this page, navigate to the settings page. And under the section security, select Secrets and variables → Actions. Here we need to create a new repository secret called `GOOGLE_PROJECT` with as value the ID of the created project. In my case, this is `secret-willow-423606-j1`

The next credential we need to get from a service account in the Google cloud console. So, for this, we need to go to the navigation menu and select `IAM and admin` → `Service accounts`. In the top bar, select `+ Create service account`. Set a `Service account ID` like `kubernetes-demo`. Press `Create and Continue` and press `Done`. 

Copy the service account email and got back into your artifact repository and select the repository by pressing the checkbox. This will open the permissions panel. Press the `Add Principal` button and past the copied service account. And assign the role of `Artifact Registry Administrator` or `Artifact Registry Writer`. After this, save the permissions and navigate back to the service account page and select the account.

Inside this account, we need to create a JSON key. Go to the `KEYS` section and press `ADD KEY` → `Create new key`. As `Key type` select JSON and created it. This will download a file that contains the key.

Next, we need to go back to the GitHub secrets and credentials and create a new repository secret called `GOOGLE_APPLICATION_CREDENTIALS` as value paste in all the contents of the downloaded JSON file.

### Result
Next we can push the code, as we have already committed it in a previous step. To do this, execute:
```terminal
git push
```

And our GitHub workflow should now start to run. We can verify this by going to the GitHub repo Actions tab and wait for the workflow to complete (or error).

## Kubernetes Engine
The next step, is to deploy our container image from the artifact registry to a Kubernetes cluster.

First, we need to create a cluster. In the Google cloud platform, select the project you want to use. Open the navigation menu and select `Kubernetes Engine` → `clusters`. This will first prompt us to enable the `Kubernetes Engine API`. On this screen, press `ENABLE` and wait for it to enable.

### Settings
On the `Kubernetes clusters` screen, we have the option to create a new cluster. For ease of use, we will use an autopilot cluster, which we might change later on or configure a new one without the autopilot.

Give a name to the cluster, I will be using `autopilot-kubernetes-demo` and change the region to the once closest to you. Leave the rest of the settings as default and press the `CREATE` button located at the bottom of the screen. Now that it is creating the cluster, we can start by creating a resource file.

### Resource
In our Python project, create a file called `resources.yaml` in the root of the files. In this file we will specify some Kubernetes resources and that we will be using our image in the artifact registry and a service.

#### Service
- **apiVersion: v1**: Specifies the version of the Kubernetes API being used.
- **kind: Service**: Indicates that this is a Service resource.
- **metadata**:
    - **name: python**: Names the Service “python”.
- **spec**:
    - **type: LoadBalancer**: Specifies that the Service is of type load balancer, which means it will be accessible from outside the Kubernetes cluster, typically through a cloud provider's load balancer.
    - **selector**:
        - **app: python**: This label selector ensures that the Service targets Pods with the label `app: python`.
    - **ports**:
        - **port: 80**: The port that the Service will expose.
        - **targetPort: 80**: The port on the container to which the traffic will be forwarded.

#### Deployment
- **apiVersion: apps/v1**: Specifies the version of the Kubernetes API for the apps group.
- **kind: Deployment**: Indicates that this is a Deployment resource.
- **metadata**:
    - **name: python**: Names the Deployment "python".
    - **labels**:
        - **app: python**: Adds a label `app: python` to the Deployment.
- **spec**:
    - **replicas: 1**: Specifies that one replica of the Pod should be running.
    - **selector**:
        - **matchLabels**:
            - **app: python**: This label selector ensures that the Deployment targets Pods with the label `app: python`.
    - **template**:
        - **metadata**:
            - **labels**:
                - **app: python**: Adds a label `app: python` to the Pod template.
        - **spec**:
            - **containers**:
                - **name: python**: Names the container "python".
                - **image: europe-west4-docker.pkg.dev/secret-willow-423606-j1/kubernetes-demo/python
                    
                    **: Specifies the container image to use.
                - **ports**:
                    - **containerPort: 80**: The port that the container will listen on.

The resource file will look as followed:
```yml
---  
apiVersion: v1  
kind: Service  
metadata:  
  name: python  
spec:  
  type: LoadBalancer  
  selector:  
    app: python  
  ports:  
  - port: 80  
    targetPort: 80  
---  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: python  
  labels:  
    app: python  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: python  
  template:  
    metadata:  
      labels:  
        app: python  
    spec:  
      containers:  
      - name: python  
        image: europe-west4-docker.pkg.dev/secret-willow-423606-j1/kubernetes-demo/python:latest  
        ports:  
        - containerPort: 80
```

### Workflow
The next step is to update our workflow file. First, we copy the `build.yml` from our previous step and update the name to `build-and-deploy.yml`. 

One of the few steps that we need to add is deploying to the GKE. First, we need to configure the kubectl client using the get-credentials command. And as final step, we need to execute kubectl apply on the resources file. 

The last step is to add the plugin for using the `get-credentials`

The updated version looks as followed:
```yml
name: Deploy to Flask to GKE  
  
on:  
  push:  
    branches:  
      - main  
  
jobs:  
  deploy:  
    runs-on: ubuntu-latest  
    steps:  
      - name: Checkout Code  
        uses: actions/checkout@v2  
  
      - name: Authenticate with Google Cloud  
        uses: 'google-github-actions/auth@v2'  
        with:  
          project_id: ${{ secrets.GOOGLE_PROJECT }}  
          credentials_json: ${{ secrets.GOOGLE_APPLICATION_CREDENTIALS }}  
  
      - name: 'Set up Cloud SDK'  
        uses: 'google-github-actions/setup-gcloud@v2'
        with:  
		  install_components: 'gke-gcloud-auth-plugin'  
  
      - name: Build and Push Docker Image  
        env:  
          GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}  
        run: |  
          gcloud auth configure-docker europe-west4-docker.pkg.dev  
          docker build -t europe-west4-docker.pkg.dev/$GOOGLE_PROJECT/kubernetes-demo/python:latest .  
          docker push europe-west4-docker.pkg.dev/$GOOGLE_PROJECT/kubernetes-demo/python:latest  
  
      - name: Deploy to GKE  
        run: |  
          gcloud container clusters get-credentials autopilot-kubernetes-demo --region europe-west4  
          kubectl apply -f resources.yml
```

Now we can commit our changes to GitHub by executing:
```terminal
git add .
git commit -m"Added resource.yml for Kubectl and update workflow"
```

### Credentials
For setting the GitHub credentials, we already have the correct credentials set in the previous step [[#Credentials]].

The only thing that we need to do is to make sure our service account that we used in the previous step also has access to the Kubernetes Engine. To do this, first copy the principle ID of the created user. Next, we need to navigate to the IAM in our Google cloud project. Press `+ GRANT ACCESS`. As principle, put the copied ID and give it the role of `Kubernetes Engine` → `Kubernetes Engine Developer`. Save the permissions.


### Result
Now push the changes and we can test the full workflow.

If everything went correctly, we should now see our application in the Kubernetes Engine → Workloads section. 

If we click on this workload and scroll all the way down. In the section called `Exposing services` we can click the URL, and we should see a functioning application.
