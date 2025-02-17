<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->


# How to run MaxText with XPK?

This document focusses on steps required to setup XPK on TPU VM and assumes you have gone through the [README](https://github.com/google/maxtext/blob/main/xpk/README.md) to understand XPK basics.

## Steps to setup XPK on TPU VM

* Verify you have these permissions for your account or service account

    Storage Admin \
    Kubernetes Engine Admin

* gcloud is installed on TPUVMs using the snap distribution package. Install kubectl using snap
```shell
sudo snap install kubectl --classic
```
* Install `gke-gcloud-auth-plugin`
```shell
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -

sudo apt update && sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```

* Authenticate gcloud installation by running this command and following the prompt
```
gcloud auth login
```

* Run this command to configure docker to use docker-credential-gcloud for GCR registries:
```
gcloud auth configure-docker
```

* Test the installation by running
```
docker run hello-world
```

* If getting a permission error, try running
```
sudo usermod -aG docker $USER
```
after which log out and log back in to the machine.

## Build Docker Image for Maxtext

1. Git clone maxtext locally

    ```shell
    git clone https://github.com/google/maxtext.git
    cd maxtext
    ```
2. Build local Maxtext docker image

    This only needs to be rerun when you want to change your dependencies. This image may expire which would require you to rerun the below command

    ```shell
    # Default will pick stable versions of dependencies
    bash docker_build_dependency_image.sh
    ```
3. Upload image to your gcp project

    This copies your working directory to the cloud and layers it on top of the dependency image. The first time you do this for a given dependency_image it will take a couple minutes. Subsequent times take less than a second!

    ```shell
    gcloud config set project $PROJECT_ID
    bash docker_upload_runner.sh CLOUD_IMAGE_NAME=${USER}_runner
    ```
4. After uploading the custom image once, xpk can handle updates to the working directory when running `xpk workload create`

    __Using XPK to upload image to your gcp project and run Maxtext__

      ```shell
      gcloud config set project $PROJECT_ID
      gcloud config set compute/zone $ZONE

      # Make sure you are in the xpk github root directory when running this command
      git clone https://github.com/google/xpk.git
      cd xpk

      python3 xpk.py workload create \
      --cluster ${CLUSTER_NAME} \
      --docker-image gcr.io/${PROJECT_ID}/${USER}_runner \
      --workload ${USER}-first-job \
      --tpu-type=v5litepod-256 \
      --num-slices=1  \
      --command "python3 MaxText/train.py MaxText/configs/base.yml base_output_directory=${BASE_OUTPUT_DIR} dataset_path=${DATASET_PATH} steps=100 per_device_batch_size=1"
      ```
