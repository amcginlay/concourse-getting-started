# concourse-getting-started

# What's the absolute minimum I have to do to get Concourse running on GCP?

## Install A Jumpbox

### Set Up Variables On Local Machine

```
GCP_PROJECT_ID=<TARGET_GCP_PROJECT_ID>
```

### Authenticate Local Machine

```
gcloud auth login
```

### Create Jumpbox VM From Local Machine

```
gcloud compute instances create "jbox-concourse" \
  --image "ubuntu-1604-xenial-v20180418" \
  --image-project "ubuntu-os-cloud" \
  --boot-disk-size "200" \
  --project "${GCP_PROJECT_ID}" \
  --zone "us-central1-a"
```

### SSH To Jumpbox

```
gcloud compute ssh ubuntu@jbox-concourse \
  --project "${GCP_PROJECT_ID}" \
  --zone "us-central1-a"
```

### Install Essential Jumpbox Tools

```
sudo apt update && sudo apt --yes install unzip make ruby

wget -O bosh https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-5.1.1-linux-amd64 && \
  chmod +x bosh && \
  sudo mv bosh /usr/local/bin/

wget -O terraform.zip https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip && \
  unzip terraform.zip && \
  sudo mv terraform /usr/local/bin

wget -O bbl https://github.com/cloudfoundry/bosh-bootloader/releases/download/v6.9.0/bbl-v6.9.0_linux_x86-64 && \
  chmod +x bbl && \
  sudo mv bbl /usr/local/bin/
```

### Authenticate Jumpbox

Follow the on-screen prompts as your execute the following:

```
gcloud auth login --quiet
```

### Setup Jumpbox Variables

Specify a bunch of variables:
```
#########################################################################################
# !!! YOU NEED TO MAKE VALUE CHANGES HERE !!!
#########################################################################################
CC_DOMAIN_NAME=pivotaledu.io     # ... or whatever you have registered for your group
CC_SUBDOMAIN_NAME=cls99env99     # ... or some ID for your environment within DOMAIN_NAME
#########################################################################################

CC_FQDN=${CC_SUBDOMAIN_NAME}.${CC_DOMAIN_NAME}
CC_APP_ROUTE=concourse.${CC_FQDN} # <--- where we expect to locate our Concourse instance
CC_PROJECT_ID=$(gcloud config get-value core/project)
CC_REGION=us-central1
```

Check the above variable assignments look as you would expect:
```
set | grep '^CC_'
```

### Configure DNS at domain level

- TODO - re-write

### Increase GCP Quotas

If necessary, increase GCP project quotas (`IAM & admin -> Quotas`) for your region as follows:

Quota type               | Quota Amount
------------------------ | ------------
CPU                      | 100
Persistent Disk SSD (GB) | 1000
In-use IP addresses      | 32

### BOSH Bootloader (BBL)

BBL will generate some files, so create a home for this operation and move there:
```
mkdir ~/bbl-concourse && cd ~/bbl-concourse
```

Create a service account for BBL
```
gcloud iam service-accounts create bbl-service-account \
  --display-name "BBL service account"
gcloud iam service-accounts keys create \
  --iam-account="bbl-service-account@$(gcloud config get-value core/project).iam.gserviceaccount.com" \
  bbl-service-account.json
gcloud projects add-iam-policy-binding $(gcloud config get-value core/project) \
  --member="serviceAccount:bbl-service-account@$(gcloud config get-value core/project).iam.gserviceaccount.com" \
  --role="roles/editor"
```

Export BBL_GCP_SERVICE_ACCOUNT_KEY and execute BBL to build Jumpbox and BOSH director VM.  
**Note** this also sets the BOSH Director's `cloud config`:
```
# if next step fails due to "too many authentication failures" 
# condsider adding "IdentitiesOnly=yes" to ~/.ssh/config
bbl up \
  --name concourse \
  --lb-type concourse \
  --iaas gcp \
  --gcp-region us-central1 \
  --gcp-service-account-key $HOME/bbl-concourse/bbl-service-account.json
```

Extract the credentials
```
eval "$(bbl print-env)"
```

### Deploy Concourse

Upload a stemcell:
```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent
```

Deploy Concourse:
```
cd ~/bbl-concourse/
CC_LB_IP=$(bbl lbs | grep '^Concourse LB' | sed 's/ //g' | cut -d':' -f2)

git clone https://github.com/concourse/concourse-deployment.git ~/bbl-concourse/concourse-deployment/
cd ~/bbl-concourse/concourse-deployment/cluster/

cat > ./bbl_ops.yml << 'EOF'
- type: replace
  path: /instance_groups/name=web/vm_extensions?/-
  value: lb
- type: replace
  path: /instance_groups/name=web/jobs/name=atc/properties/bind_port?
  value: 80
EOF

bosh deploy -n -d concourse concourse.yml \
  -l ../versions.yml \
  --vars-store ./cluster-creds.yml \
  -o ./operations/no-auth.yml \
  -o ./bbl_ops.yml \
  --var network_name=default \
  --var external_url=http://${CC_LB_IP} \
  --var web_vm_type=default \
  --var db_vm_type=default \
  --var db_persistent_disk_type=10GB \
  --var worker_vm_type=default \
  --var deployment_name=concourse
```

### Configure DNS

```
CC_FQDN_ZONE=$(echo ${CC_FQDN} | tr '.' '-')
gcloud dns managed-zones create ${CC_FQDN_ZONE} --description= --dns-name=${CC_FQDN}
gcloud dns record-sets transaction start --zone=${CC_FQDN_ZONE}
gcloud dns record-sets transaction add ${CC_LB_IP} --name=${CC_APP_ROUTE}. --ttl=60 --type=A --zone=us-central1-a
gcloud dns --project=${CC_PROJECT_ID} record-sets transaction execute --zone=us-central1-a
```
### Navigate To Concourse And Download `fly`

```
echo "http://${CC_APP_ROUTE}" <--- this is where you'll find the Concourse web UI in a browser
```

Navigate to the Concourse web UI and download the `fly` CLI utils for the OS of your local machine.

From your local machine, log-in via the `fly` CLI:
```
fly -t concourse login -c http://<CC_APP_ROUTE> # NOTE http, not https !!!
```

Now follow the [IaaS independent instructions](../shared/README.md) to create your first pipeline.

### Task Complete!
