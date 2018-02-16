# concourse-getting-started

# What's the absolute minimum I have to do to get Concourse running on GCP?

## Install

### Linux Install

The Linux instructions assume you're already SSH'd into a standard GCP Ubuntu VM residing in your target GCP project.  You should be logged on as the `ubuntu` user.  GCP Ubuntu has the `gcloud` utility included as standard.

```
sudo apt-get install unzip make ruby

sudo curl https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.48-linux-amd64 \
  -o /usr/local/bin/bosh && sudo chmod +x /usr/local/bin/bosh

curl https://releases.hashicorp.com/terraform/0.11.3/terraform_0.11.3_linux_amd64.zip \
  -o ${HOME}/terraform.zip && sudo unzip ${HOME}/terraform.zip -d /usr/local/bin/

sudo curl -L https://github.com/cloudfoundry/bosh-bootloader/releases/download/v6.1.1/bbl-v6.1.1_linux_x86-64 \
  -o /usr/local/bin/bbl && sudo chmod +x /usr/local/bin/bbl
```

### Mac Install

Install the GCP [gcloud](https://cloud.google.com/sdk/docs/quickstart-mac-os-x) utility.

Install Terraform and check version (min v0.11.1)
```
brew install terraform
terraform --version
```

Install bosh CLI v2, check version (min 2.0) and create an alias
```
brew install cloudfoundry/tap/bosh-cli
alias bosh='bosh2' # or create a symbolic link on the PATH (just as we transition from v1 to v2)
bosh --version
```

Install the BOSH Bootloader (BBL) and check version (min v5.7.3)
```
brew install cloudfoundry/tap/bbl
bbl --version
```

### Windows Install
```
1) Open Windows
1) Throw laptop out of Windows
1) Buy a computer :-)
```

## Ready to go ...

### Initialise the `gcloud` CLI

Initialise the `gcloud` CLI and follow the on-screen prompts.  If requested, you should log in with your Google Cloud Account, enter the verification code, select your target project ID (e.g. astral-chassis-194616) and an appropriate zone (e.g europe-west1-c):
```
gcloud init
```

### Setup environment variables

Specify your DOMAIN_NAME, SUBDOMAIN_NAME, FQDN, the intended URL for Concourse and PROJECT_ID:
```
#########################################################################################
# !!! YOU NEED TO MAKE VALUE CHANGES HERE !!!
#########################################################################################
CC_DOMAIN_NAME=gcp.pivotaledu.io # ... or whatever you have registered for your group
CC_SUBDOMAIN_NAME=cls99env99     # ... or some ID for your environment within DOMAIN_NAME
#########################################################################################

CC_FQDN=${CC_SUBDOMAIN_NAME}.${CC_DOMAIN_NAME}
CC_URL=concourse.${CC_FQDN}
CC_PROJECT_ID=$(gcloud config get-value core/project)
CC_REGION=$(gcloud config get-value compute/region)
```

Check these variables look as you would expect:
```
set | grep '^CC_'
```

### Configure DNS

Run the following `host` check if your environment can be reached from the internet
```
if host ${CC_FQDN}; then echo SUCCESS; else echo FAIL; fi
```

If the above command yields **SUCCESS**, we're done configuring DNS for now and you should skip to the next section.

If the above command yields **FAIL**, you should first check to see that the hosted zone (Cloud DNS) for `${DOMAIN_NAME}` includes an "A" type Record Set for `${FQDN}` which specifies **ALL** the Google DNS servers.  If not, you should add one.  Assuming your DNS configuration is done in GCP, the required sequence of commands could look something like this:
```
#########################################################################################
# !!! YOU NEED TO MAKE VALUE CHANGES HERE !!!
#########################################################################################
# Specify the Cloud DNS Zone which manages CC_DOMAIN_NAME
CC_DOMAIN_NAME_ZONE=$(echo ${CC_DOMAIN_NAME} | tr '.' '-')        # <--- for example

# Specify the Project ID whhere CC_DOMAIN_NAME_ZONE is maintained
CC_CONFIG_PROJECT_ID=cso-education-shared                         # <--- for example
#########################################################################################

gcloud dns --project=${CC_CONFIG_PROJECT_ID} record-sets transaction start --zone=${CC_DOMAIN_NAME_ZONE}

gcloud dns --project=${CC_CONFIG_PROJECT_ID} \
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
  --name=${CC_FQDN} \
  --ttl=60 --type=NS --zone=${CC_DOMAIN_NAME_ZONE}

gcloud dns --project=${CC_CONFIG_PROJECT_ID} record-sets transaction execute --zone=${CC_DOMAIN_NAME_ZONE}
```

Configure the DNS zone in your target project to complete the linkage:
```
gcloud dns --project=${CC_PROJECT_ID} managed-zones create $(echo ${CC_FQDN} | \
  tr '.' '-') --description= --dns-name=${CC_FQDN}
```

You should not proceed until the `host` check (repeated below) yields **SUCCESS**
```
if host ${CC_FQDN}; then echo SUCCESS; else echo FAIL; fi
```

### Increase GCP Quotas

If necessary, increase GCP project quotas (`IAM & admin -> Quotas`) for your region as follows:

Quota type               | Quota Amount
------------------------ | ------------
CPU                      | 100
Persistent Disk SSD (GB) | 1000
In-use IP addresses      | 32

### BOSH Bootloader (BBL)

BBL will generate some files, so create a home for this operation
```
mkdir ${HOME}/bbl-concourse && cd ${HOME}/bbl-concourse
```

Create a service account for BBL and export BBL_GCP_SERVICE_ACCOUNT_KEY (required by BBL)
```
gcloud iam service-accounts create bbl-service-account \
  --display-name "BBL service account"
gcloud iam service-accounts keys create \
  --iam-account="bbl-service-account@${CC_PROJECT_ID}.iam.gserviceaccount.com" \
  bbl-service-account.json
gcloud projects add-iam-policy-binding ${CC_PROJECT_ID} \
  --member="serviceAccount:bbl-service-account@${CC_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"

export BBL_GCP_SERVICE_ACCOUNT_KEY=$(cat $HOME/bbl-concourse/bbl-service-account.json)
```

Execute BBL to build Jumpbox and BOSH director VM
```
bbl up --iaas gcp --name concourse --gcp-region ${CC_REGION} --lb-type concourse
# if next step fails due to "too many authentication failures" 
# condsider adding "IdentitiesOnly=yes" to ${HOME}/.ssh/config
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

Configure the DNS entries 
(**Note** the above `host` check must have already yielded `SUCCESS`)
```
cd ${HOME}/bbl-concourse
export EXT_IP=$(bbl lbs | grep "^Concourse LB:" | cut -d":" -f2 | xargs)
gcloud dns record-sets transaction start --zone=${SUBDOMAIN_NAME}
gcloud dns record-sets transaction add ${EXT_IP} \
  --name=${CONCOURSE_URL}. --ttl=300 --type=A --zone=${SUBDOMAIN_NAME}
gcloud dns record-sets transaction execute --zone=${SUBDOMAIN_NAME}
```

Wait for DNS lookup to yield an IP address - this may take a few mins

```
dig +short ${CONCOURSE_URL}
```

Check the Concourse web UI and download the `fly` CLI utils
```
open http://${CONCOURSE_URL}
curl -L "http://${CONCOURSE_URL}/api/v1/cli?arch=amd64&platform=darwin" \
  -o /usr/local/bin/fly
```

Log-in via the `fly` CLI
```
fly -t concourse login -c http://${CONCOURSE_URL}/
```

Now follow the [IaaS independent instructions](../shared/README.md) to create your first pipeline.

### Task Complete!
