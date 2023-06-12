---
title: Deploying a Flask App to Kubernetes on AWS - Part 1
date: 2020-06-11 08:15:00
categories:
- [web, devops, python, flask]
tags:
- python
- flask
- aws
- k8s
---

## Introduction

When I set myself to the task of migrating my current organization’s internal CI/CD pipelines to Github Actions, I realized very quickly that there were very limited and scattered resources on the internet teaching about deploying applications to App Engine. Thus, I put together this general guide.

Later on, I realized that Github Actions provides an `action` (equivalent of a Circle CI orb) for deploying to App Engine, which eliminates a lot of boilerplate I’d written in the deployment manifest. You can checkout the action <a rel="nofollow noopener" target="_blank" href="https://github.com/google-github-actions/deploy-appengine"> here </a>.

## Prerequisites

You need the following to get started:

- `OpenSSL` → Required for encryption/decryption of manifests/files. You can download it on the following links:

```bash
# windows: https://sourceforge.net/projects/openssl-for-windows/

# linux: https://www.howtoforge.com/tutorial/how-to-install-openssl-from-source-on-linux/

# mac: install via brew
brew install openssl
```

- Pwgen → a bash utility used with openssl for encyption/decryption tasks.

```bash
# windows: https://pwgen-win.sourceforge.io/download/

# linux: https://snapcraft.io/pwgen-tyhicks

# mac: install via brew
brew install pwgen
```

- `Git` → for version control
- `base64` → A bash utility for base64 encoding/decoding files.
- A Github account and Project

## Steps

The first step is to generate and export the encryption key that we’ll use to encrypt our secrets:

```bash
export ENCRYPTION_KEY=$(pwgen 32 1)
```

`pwgen` generates a 32 char long secure and random password.

To check that the encryption key is in your environment, echo the key as follows:

```bash
$ echo $ENCRYPTION_KEY

# mohvishiebup0iashaideij5EebaeSae
```

This done, you can now proceed to encrypt each of your credential files as follows. Make sure to run this command for each of your files:

```bash
openssl aes-256-cbc -in CREDENTIAL_FILE  -out CREDENTIAL_FILE.enc  -pass pass:$ENCRYPTION_KEY -e -md sha1
```

With each of your files encrypted, you can proceed to create the Github action workflow file as follows:

1. In the root of your project directory, create a `.github/workflows` directory and inside it a `workflow.yaml` file.

```bash
mkdir .github $$ cd .github $$ mkdir workflows $$ cd workflows $$ touch workflow.yaml
```

2. In the Google Cloud console for your project, generate and download the service account file with the following permissions:

```bash
App Engine Deployer
App Engine Service Admin
Cloud Build Editor
Storage Object Creator
Storage Object Viewer
```

Also make sure that the App Engine Admin API is enabled.

3. Encode the service account file in base64 as follows, and copy the resulting output:

```bash
base64 service_account.json
```

4. On your Github project repo, navigate to `Settings` → `Secrets` and create an action secret with the key `GCLOUD_AUTH` and the value being the base64 encoded string. Also add secrets for the App Engine project name and `ENCRYPTION_KEY` for the encryption key used with openssl. Your dashboard in Github should look like below:

![Github Secrets Page](/images/deploying-flask-app-to-k8s-aws-part-1/github_secrets_page.png)


You can now proceed to write your workflow manifest. Below is a sample manifest currently used for the Glue code. Each step of the manifest has been explained. You can use this as a template for your manifest:

```yaml
# .github/workflows/workflow.yaml

name: Deploy to App Engine # this is the name of the workflow
on:
  push:
    branches: [ master ] # this workflow will activate when a push is made to the master branch
jobs: # these are the tasks to be executed in the workflow
  deploy: # this is a sub-task
    runs-on: ubuntu-latest # the os to run the sub-task in
    steps: # these are individual steps of a sub-task
      - uses: actions/checkout@v2 # checks out to the build directory

      - name: Decrypt secrets # handles decryption of the secrets you encrypted locally with openssl
        env:
          ENCRYPTION_KEY: ${{ secrets.ENCRYPTION_KEY }}
        run: |
          openssl aes-256-cbc -in env_variables.yaml.enc -out env_variables.yaml -pass pass:$ENCRYPTION_KEY -d -md sha1
          openssl aes-256-cbc -in gcloud_credentials.json.enc -out gcloud_credentials.json -pass pass:$ENCRYPTION_KEY -d -md sha1

      - name: Setup gcloud # sets up your gcloud client
        uses: google-github-actions/setup-gcloud@v0.2.0
        env:
          service_account_key: ${{ secrets.GCLOUD_AUTH }}
          project_id: ${{ secrets.APP_ENGINE_PROJECT_NAME }}
          export_default_credentials: true

      - name: Deploy to App Engine # handles the deployment to App Engine
        env:
          service_account_key: ${{ secrets.GCLOUD_AUTH }}
          project_id: ${{ secrets.APP_ENGINE_PROJECT_NAME }}
        run: |
          echo $service_account_key | base64 --decode >> account.json
          gcloud auth activate-service-account --key-file account.json
          gcloud config set project $project_id
          gcloud app deploy -q --verbosity=debug
```

With this setup, you should be able to trigger a successful build to App Engine via Github Actions. You can monitor your build on the Actions tab of your project repository on Github:

![Github Actions Page](/images/deploying-flask-app-to-k8s-aws-part-1/github_actions_page.png)
