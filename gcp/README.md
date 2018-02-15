# concourse-getting-started

# What's the absolute minimum I have to do to get Concourse running on GCP?

## Install

### Linux Install

These instructions assume you're SSH'd into an Ubuntu jumpbox in your target GCP project as the `ubuntu` user.  GCP Ubuntu installs _should_ have the GCP `gcloud` utility included as standard.

```
sudo apt-get install unzip make ruby

sudo curl https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.48-linux-amd64 -o /usr/local/bin/bosh
sudo chmod +x /usr/local/bin/bosh

curl https://releases.hashicorp.com/terraform/0.11.3/terraform_0.11.3_linux_amd64.zip -o ${HOME}/terraform.zip
sudo unzip ${HOME}/terraform.zip -d /usr/local/bin/

sudo curl -L https://github.com/cloudfoundry/bosh-bootloader/releases/download/v6.1.1/bbl-v6.1.1_linux_x86-64 -o /usr/local/bin/bbl
sudo chmod +x /usr/local/bin/bbl
```

### Mac Install

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

## Ready to go ...

Specify your DOMAIN_NAME, CONCOURSE_URL and PROJECT_ID
```
export ENV_NAME=<DISTINCT_INSTANCE_NAME>
export DOMAIN_NAME=gcp.pivotaledu.io # ... or whatever you have registered
export CONCOURSE_URL=concourse.${ENV_NAME}.${DOMAIN_NAME}
export PROJECT_ID=$(gcloud config get-value core/project)
```

Run the following `host` check if your environment can be reached from the internet
```
if host ${ENV_NAME}.${DOMAIN_NAME}; then echo SUCCESS; else echo FAIL; fi
```

If the above command yields **FAIL**, you should check to see that the hosted zone (Cloud DNS) for `${DOMAIN_NAME}` includes an "A" type Record Set for `${ENV_NAME}.${DOMAIN_NAME}` which specifies **ALL** the Google DNS servers.  If not, you should add one.  Assuming all DNS configuration is done in GCP, the required sequence of commands could look something like this:
```
DOMAIN_NAME_ZONE=<CLOUD DNS ZONE for DOMAIN_NAME>
CONFIG_PROJECT_ID=<HOST PROJECT ID OF DOMAIN_NAME_ZONE>

gcloud dns --project=${CONFIG_PROJECT_ID} record-sets transaction start --zone=${DOMAIN_NAME_ZONE}

gcloud dns --project=${CONFIG_PROJECT_ID} \
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
  --ttl=60 --type=NS --zone=${DOMAIN_NAME_ZONE}

gcloud dns --project=${CONFIG_PROJECT_ID} record-sets transaction execute --zone=${DOMAIN_NAME_ZONE}
```

Configure the DNS zone in your target project to complete the linkage:
```
gcloud dns --project=${PROJECT_ID} managed-zones create $(echo ${ENV_NAME}.${DOMAIN_NAME} | tr '.' '-') --description= --dns-name=${ENV_NAME}.${DOMAIN_NAME}
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
export BBL_GCP_SERVICE_ACCOUNT_KEY=$(cat $HOME/bbl-concourse/bbl-service-account.json)
```

Execute BBL to build Jumpbox and BOSH director VM
```
bbl up --iaas gcp --name my-concourse --gcp-region ${REGION} --lb-type concourse
# if next step fails due to "too many authentication failures" condsider adding "IdentitiesOnly=yes" to ${HOME}/.ssh/config
```

Extract the credentials
```
eval "$(bbl print-env)"
```

Upload a stemcell and deploy Concourse
```
export external_url=http://$(bbl lbs |awk -F':' '{print $2}' |sed 's/ //')

bosh upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent

git clone https://github.com/concourse/concourse-deployment.git

cd concourse-deployment/cluster
cat > bbl_ops.yml << 'EOF'
- type: replace
  path: /instance_groups/name=web/vm_extensions?/-
  value: lb
- type: replace
  path: /instance_groups/name=web/jobs/name=atc/properties/bind_port?
  value: 80
EOF

bosh deploy -d concourse concourse.yml \
  -l ../versions.yml \
  --vars-store cluster-creds.yml \
  -o operations/no-auth.yml \
  -o bbl_ops.yml \
  --var network_name=default \
  --var external_url=$external_url \
  --var web_vm_type=default \
  --var db_vm_type=default \
  --var db_persistent_disk_type=10GB \
  --var worker_vm_type=default \
  --var deployment_name=concourse
```

Configure the DNS entries (**Note** the above `host` check must have already yielded `SUCCESS`)
```
cd ${HOME}/bbl-concourse
export EXT_IP=$(bbl lbs | grep "^Concourse LB:" | cut -d":" -f2 | xargs)
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
fly -t concourse login -c http://${CONCOURSE_URL}/
```

Now follow the [IaaS independent instructions](../shared/README.md) to create your first pipeline.

### Task Complete!
