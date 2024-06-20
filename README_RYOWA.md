# Table of Contents
- [Table of Contents](#table-of-contents)
- [Custom Configuration](#custom-configuration)
  - [Breif](#breif)
  - [Postgres Database](#postgres-database)
  - [Persistent Cloud Storage](#persistent-cloud-storage)
    - [Setup GCS ref](#setup-gcs-ref)
      - [Create GCS Buket](#create-gcs-buket)
      - [Configuretion Cors](#configuretion-cors)
      - [Confiuretion GCS bucket](#confiuretion-gcs-bucket)
  - [Application setting](#application-setting)
  - [Combination all environments](#combination-all-environments)
  - [Docker run command](#docker-run-command)
    - [Docker image url:](#docker-image-url)
  - [Download file from Google Cloud Storage](#download-file-from-google-cloud-storage)

# Custom Configuration

⚠️ All of the contents in this file regarding configurations have been gathered from public documentation, Slack, GitHub issues, and Stack Overflow.

⚠️ It does not contain any sensitive data or credential knowledge and intends to serve as a guidebook.

## Breif
label-studio default setting config will use internal database, save images in local storage. Hence, when deployed on VM all of data is stay in just 1 place.

wihch is dangerouse if VM has compromised, or unexpected incident has happended and all data has been lost.

## Postgres Database
```yaml
# ==============================
# Database
# ==============================
# Setup connection Postgresql
DJANGO_DB=default # have to be exactly "default"
POSTGRE_NAME={{DATABASE_NAME}}
POSTGRE_USER={{USERNAME}}
POSTGRE_PASSWORD={{PASSWORD}}
POSTGRE_PORT=5432
POSTGRE_HOST={{IP_HOST}}
# DATABASE_URL=postgresql://{{USERNAME}}:{{PASSWORD}}@{{IP_HOST}}:5432/{{DATABASE_NAME}}
```

## Persistent Cloud Storage
[guide-persistent_storage](https://labelstud.io/guide/persistent_storage)

In this guide we use `Google Cloud Storage` or `GCS` for keep the image data.
**presistent cloud storage**: automatically `save`, `fetch` image in **cloud** instead local storage
### Setup GCS [ref](https://labelstud.io/guide/persistent_storage#Set-up-Google-Cloud-Storage)
Set up Google Cloud Storage (GCS) as the persistent storage for Label Studio hosted in Google Cloud Platform (GCP) or Docker Compose.
#### Create GCS Buket
1. create **bucket** *sample*: `sample-bucket-1234`. See [Creating storage buckets](https://cloud.google.com/storage/docs/creating-buckets) in the Google Cloud Storage guide.
1. choose **uniform access control** for [access control method for the bucket](https://cloud.google.com/storage/docs/access-control)
1. create **IAM Service Account**. See [Creating and managing service accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts) in the Google Cloud Storage guide.
1. Select the predefined **Storage Object Admin** IAM role to add to the service account so that the account can create, access, and delete objects in the bucket.
1. Add a **condition** to the role that restricts the role to access only objects that belong to the bucket you created.:
    - Select Add Condition when setting up the service account IAM role, then use the Condition Builder to specify the following values:
        - Condition type: `Name`
        - Operator: `Starts with`
        - Value: `projects/_/buckets/sample-bucket-1234`
1. Select the predefined *Service Account* ➡️ **serviceAccountTokenCreator**  and don't add any condition.
#### Configuretion Cors
Set up CORS access to your bucket. See [Configuring cross-origin resource sharing (CORS)](https://cloud.google.com/storage/docs/configuring-cors#configure-cors-bucket) in the Google Cloud User Guide. Use or modify the following example:
```json
// cors-config.json
   {
      "origin": ["*"],
      "method": ["GET","PUT","POST","DELETE","HEAD"],
      "responseHeader": ["Content-Type","Access-Control-Allow-Origin"],
      "maxAgeSeconds": 3600
   }
```
Replace `YOUR_BUCKET_NAME` with your actual bucket name in the following command to update CORS for your bucket:
```bash
gcloud storage buckets update gs://BUCKET_NAME --cors-file=CORS_CONFIG_FILE
```
View the CORS configuration for a bucket
```bash
gcloud storage buckets describe gs://BUCKET_NAME --format="default(cors_config)"
```
#### Confiuretion GCS bucket
- First, we have to download `credential.json` from our IAM that has created
- Create a service account key from the UI and download the JSON. Follow the steps for [Creating and managing service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) in the Google Cloud Identity and Access Management guide.
- After downloading the JSON for the service account key, update or create references to the JSON, your projectID, and your bucket in your env.list file. Optionally, you can choose a folder by specifying `STORAGE_GCS_FOLDER` (default is "" or omit this argument):
```yaml
# ==============================
# Cloud Storage
# https://labelstud.io/guide/persistent_storage#Set-up-Google-Cloud-Storage
# https://app.slack.com/client/TQ8AKDJ6T/search
# ==============================
STORAGE_TYPE=gcs
STORAGE_GCS_BUCKET_NAME={{GCS_BUCKET_NAME}}
STORAGE_GCS_PROJECT_ID={{PROJECT_ID}}
# STORAGE_GCS_FOLDER can be leave as omit
STORAGE_GCS_FOLDER=
# GOOGLE_APPLICATION_CREDENTIALS is the path in docker container; when use we simply volumn mount the with the path of host 
GOOGLE_APPLICATION_CREDENTIALS=/opt/heartex/secrets/key.json
```
- Have to mount volumn in docker 
```yaml
-v credential.json:/opt/heartex/secrets/key.json:ro
```
## Application setting 
It is intend to chage some config of `label-studio` 
- from this [github issue](https://github.com/HumanSignal/label-studio/issues/4317#issuecomment-1902666226) about **Persistent Storage on GCS not working on Imports** or means not display..
  ```python
  # label_studio/core/feature_flags/stale_feature_flags.py
  ff_back_dev_2915_storage_nginx_proxy_26092022_short=False 
  ```

- This is environment variables that need to be set
```yaml
# ==============================
# Secure Label Studio:
# https://labelstud.io/guide/security
# ==============================
# Setup user cannot create account without invitation link
LABEL_STUDIO_DISABLE_SIGNUP_WITHOUT_LINK=true
LABEL_STUDIO_USERNAME={{EMAIL_DEFAULT_USER}}
LABEL_STUDIO_PASSWORD={{PASSWORD_DEFAULT_USER}}
SSRF_PROTECTION_ENABLED=true
COLLECT_ANALYTICS=false
```


## Combination all environments
- in `.env.list`
```yaml
# ==============================
# Database
# ==============================
# Setup connection Postgresql
DJANGO_DB=default # have to be exactly "default"
POSTGRE_NAME={{DATABASE_NAME}}
POSTGRE_USER={{USERNAME}}
POSTGRE_PASSWORD={{PASSWORD}}
POSTGRE_PORT=5432
POSTGRE_HOST={{IP_HOST}}
# DATABASE_URL=postgresql://{{USERNAME}}:{{PASSWORD}}@{{IP_HOST}}:5432/{{DATABASE_NAME}}
# ==============================
# Cloud Storage
# https://labelstud.io/guide/persistent_storage#Set-up-Google-Cloud-Storage
# https://app.slack.com/client/TQ8AKDJ6T/search
# ==============================
STORAGE_TYPE=gcs
STORAGE_GCS_BUCKET_NAME={{GCS_BUCKET_NAME}}
STORAGE_GCS_PROJECT_ID={{PROJECT_ID}}
# STORAGE_GCS_FOLDER can be leave as omit
STORAGE_GCS_FOLDER=
# GOOGLE_APPLICATION_CREDENTIALS is the path in docker container; when use we simply volumn mount the with the path of host 
GOOGLE_APPLICATION_CREDENTIALS=/opt/heartex/secrets/key.json
# ==============================
# Secure Label Studio:
# https://labelstud.io/guide/security
# ==============================
# Setup user cannot create account without invitation link
LABEL_STUDIO_DISABLE_SIGNUP_WITHOUT_LINK=true
LABEL_STUDIO_USERNAME={{EMAIL_DEFAULT_USER}}
LABEL_STUDIO_PASSWORD={{PASSWORD_DEFAULT_USER}}
SSRF_PROTECTION_ENABLED=true
COLLECT_ANALYTICS=false
```

## Docker run command
The `thaiteam/label-studio:{{TAG_VERSION}}` came from manual build of this [Config](#application-setting) `ff_back_dev_2915_storage_nginx_proxy_26092022_short=False`

For reference the docker image that changed `ff_back_dev_2915_storage_nginx_proxy_26092022_short=False` which is *hard code* will be pushed in docker hub. Using **thaiteam.ryowa@gmail.com** as owner.

- `--env-file` equivalent to `-e KEY=VALUE`  but just read environtments from file.

- Windows powershell
```powershell
docker run -it -p 9001:8080 `
--env-file .env.list `
-v ${pwd}/credential.json:/opt/heartex/secrets/key.json `
{{USERNAME}}/{{IMAGE_NAME}}:{{TAG_VERSION}} label-studio --log-level DEBUG
```
- Linux shell
```bash
docker run -it -p 9001:8080 \
--env-file .env.list \
-v %cd%/credential.json:/opt/heartex/secrets/key.json \
{{USERNAME}}/{{IMAGE_NAME}}:{{TAG_VERSION}} label-studio --log-level DEBUG
```
### Docker image url: 
https://hub.docker.com/r/thaiteam/label-studio/tags

## Download file from Google Cloud Storage
- Windows powershell
```
gsutil -m cp -r `
  "gs://BUCKET_NAME/PREFIX_FOLDER" `
  .
```
- Linux shell
```
gsutil -m cp -r \
  "gs://BUCKET_NAME/PREFIX_FOLDER" \
  .
```

Last edited: 2024/06/21 08:11; Unif