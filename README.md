# gcp-flask-demo
Demo deploying Flask simple app in GCP
Based on: https://cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-python-service

## GCP Console

1. Enable Cloud Run API (When asked, click Authorize)

```bash
gcloud services enable run.googleapis.com
```

Output example:
```bash
jvalenzani@cloudshell:~ (vigilant-garbanzo)$ gcloud services enable run.googleapis.com
Operation "operations/acf.p2-144305492479-afff0b4c-e30b-40c2-8479-5e053598feef" finished successfully.
```

Set Compute Region
```bash
jvalenzani@coudshell:~ (vigilant-garbanzo)$ gcloud config set compute/region us-central1
Updated property [compute/region].
jvalenzani@cloudshell:~ (vigilant-garbanzo)$ LOCATION="us-central1"
```


2. Create / Pull your application

app/app.py
```python
import os

from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello_world():
    """Example Hello World route."""
    name = os.environ.get("NAME", "World")
    return f"Hello {name}!"


if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

NOTE: This demo will require `Flask` and `gUnicorn` packages.
You can install them with `pip` to test

```
pip install flask gunicorn
```

You can create requirements.txt from `pip` if you're using a `venv`.

```bash
pip freeze > requirements.txt
```
NOTE: Don't use it on your common environment or will be populated with every package on your system. :)

You'll only need Flask and gunicorn:
```
Flask==2.3.3
gunicorn==21.2.0
```

3. Create the Dockerfile

```Dockerfile
# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.11-slim

# Allow statements and log messages to immediately appear in the logs
ENV PYTHONUNBUFFERED True

# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./

# Install production dependencies.
RUN pip install --no-cache-dir -r requirements.txt

# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
# Timeout is set to 0 to disable the timeouts of the workers to allow Cloud Run to handle instance scaling.
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 my_app:app
```

4. Add `.dockerignore` file to exclude files from the image

```
Dockerfile
README.md
*.pyc
*.pyo
*.pyd
__pycache__
.pytest_cache
```

5. Build your Image

Cloud Build is a service that executes your builds on GCP. It executes a series of build steps, where each build step is run in a Docker container to produce your application container (or other artifacts) and push it to Cloud Registry, all in one command.

Once pushed to the registry, you will see a SUCCESS message containing the image name (gcr.io/[PROJECT-ID]/my_app). The image is stored in Artifact Registry and can be re-used if desired.

```bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/my_app
```

List container images associated with the current project.
```bash
gcloud container images list
```

6. Test locally on Cloud Shell

docker run -d -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld

7. Deploy to Cloud Run

gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld --allow-unauthenticated --region=$LOCATION

