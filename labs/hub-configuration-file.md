# Create a Hub Configuration File

In this section you will generate a [hub configuration file](https://hub.github.com/hub.1.html#CONFIGURATION) to hold the GitHub credentials generated in the [prerequisites lab](prerequisites.md#generate-a-github-api-token). The hub configuration file is used by the `hub` command when automating GitHub related tasks.

## Generate the Hub Configuration File

Create a hub configuration file in the current directory:

```
cat <<EOF > hub
github.com:
  - protocol: https
    user: ${GITHUB_USERNAME}
    oauth_token: $(cat .pipeline-tutorial-github-api-token)
EOF
```

Set the `HUB_CONFIG` environment variable to point to the generated hub configuration file:

```
HUB_CONFIG="${PWD}/hub"
```

## Encrypt the Hub Configuration File and Upload to Google Cloud Storage

In this section you will encrypt the hub configuration file using the [Google Key Management Service](https://cloud.google.com/kms) (KMS) and upload the encrypted file to a [Google Cloud Storage](https://cloud.google.com/storage) (GCS) bucket. Preforming these tasks will make the hub configuration file securely available to any automated build steps performed by Cloud Build.

### Create a KMS Keyring and Encryption Key

A KMS keyring and encryption key is required to encrypt the hub configuration file.

Create a new keyring:

```
gcloud kms keyrings create pipeline --location=global
```

Generate a new encryption key:

```
gcloud kms keys create github \
  --location=global \
  --keyring=pipeline \
  --purpose=encryption
```

Encrypt the hub configuration file using the `pipeline` keyring and the `github` encryption key:

```
gcloud kms encrypt \
  --plaintext-file "${HUB_CONFIG}" \
  --ciphertext-file hub.enc \
  --location=global \
  --keyring=pipeline \
  --key=github
```

The encrypted hub configuration file `hub.enc` is now available in the current directory.

### Upload the Encrypted Hub Configuration File to GCS

In this section you will create a GCS bucket and upload the encrypted hub configuration file to it.

Retrieve the active GCP project ID and store it in the `PROJECT_ID` env var:

```
PROJECT_ID=$(gcloud config get-value core/project)
```

Create a GCS bucket that will hold the encrypted hub configuration file:

```
gsutil mb gs://${PROJECT_ID}-pipeline-configs
```

Upload the encrypted hub configuration file to the pipeline configs GCS bucket:

```
gsutil cp hub.enc gs://${PROJECT_ID}-pipeline-configs/
```

### Grant the Cloud Build Service Account Access to the GitHub Encryption Key

In this section you will grant access to the `github` encrypt key to the Cloud Build service account. Performing these steps will enable Cloud Build to decrypt the hub configuration file during any automated build.

Retrieve the active GCP project number and store it in the `PROJECT_NUMBER` env var:

```
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
```

Grant the Cloud Build service account access to the `github` encryption key:

```
gcloud kms keys add-iam-policy-binding github \
  --location=global \
  --keyring=pipeline \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
```

Next: [Setup the GitHub Repositories](github-repositories.md)
