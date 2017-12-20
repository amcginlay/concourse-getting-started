# concourse-getting-started
### What's the least I have to do to get Concourse running?

[Mac only]

Install the GCP [gcloud](https://cloud.google.com/sdk/docs/quickstart-mac-os-x) utility.

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

Specify your domain name and environment
```
export ENV_NAME=<DISTINCT_INSTANCE_NAME>
export DOMAIN_NAME=gcp.pivotaledu.io # ... or whatever you have registered
export CONCOURSE_URL=concourse.${ENV_NAME}.${DOMAIN_NAME}
```

Run the following `host` check if your environment can be reached from the internet
```
if host ${ENV_NAME}.${DOMAIN_NAME}; then echo SUCCESS; else echo FAIL; fi
```

If the above check yields **FAIL**, you should modify the parent DNS zone to include an "A" type Record Set entry specifying **ALL** the Google DNS servers.  For instance if your parent zone for `${DOMAIN_NAME}` is named `<PARENT_ZONE>` and is registered within `<PROJECT_ID>`, the required sequence of commands could look something like this:
```
gcloud dns --project=<PROJECT_ID> record-sets transaction start --zone=<PARENT_ZONE>

gcloud dns --project=<PROJECT_ID> \
  record-sets transaction add \
  ns-cloud-a1.googledomains.com. \
  ns-cloud-b1.googledomains.com. \
  ns-cloud-c1.googledomains.com. \
  ns-cloud-d1.googledomains.com. \
  ns-cloud-e1.googledomains.com. \
  ns-cloud-a2.googledomains.com. \
  ns-cloud-b2.googledomains.com. \
  ns-cloud-c2.googledomains.com. \
  ns-cloud-d2.googledomains.com. \
  ns-cloud-e2.googledomains.com. \
  ns-cloud-a3.googledomains.com. \
  ns-cloud-b3.googledomains.com. \
  ns-cloud-c3.googledomains.com. \
  ns-cloud-d3.googledomains.com. \
  ns-cloud-e3.googledomains.com. \
  ns-cloud-a4.googledomains.com. \
  ns-cloud-b4.googledomains.com. \
  ns-cloud-c4.googledomains.com. \
  ns-cloud-d4.googledomains.com. \
  ns-cloud-e4.googledomains.com. \
  --name=${ENV_NAME}.${DOMAIN_NAME}. \
  --ttl=60 --type=NS --zone=<PARENT_ZONE>

gcloud dns --project=<PROJECT_ID> record-sets transaction execute --zone=<PARENT_ZONE>
```

**Note** you should not proceed until the above `host` check yields **SUCCESS**.

BBL will generate some files, so create a home for this operation
```
mkdir ${HOME}/bbl-concourse && cd ${HOME}/bbl-concourse
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
  -v "system_domain=${CONCOURSE_URL}""
```

Grab the Concourse username and password - keep them to hand for use with the webpage
```
export CONCOURSE_USERNAME=$(bosh interpolate concourse-vars.yml --path /basic_auth_username)
export CONCOURSE_PASSWORD=$(bosh interpolate concourse-vars.yml --path /basic_auth_password)
echo ${CONCOURSE_USERNAME} ${CONCOURSE_PASSWORD}
```

Configure the DNS entries (**Note** the above `host` check must have already yielded `SUCCESS`)
```
export EXT_IP=$(bbl lbs | grep "^Concourse LB:" | cut -d":" -f2 | xargs)
gcloud dns managed-zones create ${ENV_NAME} --description= --dns-name=${ENV_NAME}.${DOMAIN_NAME}.
gcloud dns record-sets transaction start --zone=${ENV_NAME}
gcloud dns record-sets transaction add ${EXT_IP} \
  --name=${CONCOURSE_URL}. --ttl=300 --type=A --zone=${ENV_NAME}
gcloud dns record-sets transaction execute --zone=${ENV_NAME}
```

Wait for DNS lookup to yield an IP address - this may take a few mins

```
dig +short ${CONCOURSE_URL}
```

Check the Concourse web UI and download the `fly` CLI utils
```
open http://${CONCOURSE_URL}
curl -L "http://${CONCOURSE_URL}/api/v1/cli?arch=amd64&platform=darwin" -o /usr/local/bin/fly
```

Log-in via the `fly` CLI
```
fly -t concourse login -c http://${CONCOURSE_URL}/ -u ${CONCOURSE_USERNAME} -p ${CONCOURSE_PASSWORD}
```

### Task Complete!
