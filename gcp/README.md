# concourse-getting-started
### What's the least I have to do to get Concourse running?

Install bosh CLI v2, check version (min 2.0) and create an alias

```
brew install cloudfoundry/tap/bosh-cli
bosh2 --version
alias bosh='bosh2'
```

Install Terraform and check version (min v0.11.1)

```
brew install terraform
terraform --version
```

Install the BOSH Bootloader (BBL) and check version (min v5.7.3)

```
brew install bbl
bbl --version
```

BBL will generate some files, so create a home for this operation

```
mkdir $HOME/bbl-concourse && cd $HOME/bbl-concourse
```

Initialise `gcloud` to target your project and region

```
gcloud init # and follow the prompts
export PROJECT_ID=$(gcloud config get-value core/project)
export REGION=$(gcloud config get-value compute/region)
```

Increase GCP project quotas for your region as follows:

Quota type               | Quota Amount
------------------------ | ------------
CPU                      | 100
Persistent Disk SSD (GB) | 1000
In-use IP addresses      | 32

Create a service account for BBL and capture the settings
```
gcloud iam service-accounts create bbl-service-account \
  --display-name "BBL service account"
gcloud iam service-accounts keys create \
  --iam-account="bbl-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  bbl-service-account.json
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bbl-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"
export BBL_GCP_SERVICE_ACCOUNT_KEY=$(cat $HOME/bbl/bbl-service-account.json)
```

Execute BBL to build Jumpbox and BOSH director VM, then extract credentials
```
bbl up --iaas gcp --name my-concourse --gcp-region ${REGION} --lb-type concourse
# if next step fails due to "too many authentication failures" condsider adding "IdentitiesOnly=yes" to ${HOME}/.ssh/config
eval "$(bbl print-env)"
```

Upload a stemcell
```
wget https://s3.amazonaws.com/bosh-gce-light-stemcells/light-bosh-stemcell-3468.15-google-kvm-ubuntu-trusty-go_agent.tgz
bosh upload-stemcell light-bosh-stemcell-3468.15-google-kvm-ubuntu-trusty-go_agent.tgz
```

Deploy Concourse
```
wget https://github.com/evanfarrar/concourse-deployment/archive/v0.0.2.tar.gz && tar xvf v0.0.2.tar.gz
bosh deploy -n concourse-deployment-0.0.2/concourse-deployment.yml \
  -d concourse \
  --vars-store concourse-vars.yml \
  -v "system_domain=concourse.${DOMAIN}"
```

Grab the Concourse username and password
```
export CONCOURSE_USERNAME=$(bosh interpolate concourse-vars.yml --path /basic_auth_username)
export CONCOURSE_PASSWORD=$(bosh interpolate concourse-vars.yml --path /basic_auth_password)
echo ${CONCOURSE_USERNAME} ${CONCOURSE_PASSWORD} # keep them to hand for use with the webpage
```


### Task Complete!
