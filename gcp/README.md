# concourse-getting-started

# What's the absolute minimum I have to do to get Concourse running on GCP?

## Install A Jumpbox

### Set Up Variables On Local Machine

```bash
GCP_PROJECT_ID=<TARGET_GCP_PROJECT_ID>
```
Note that this is the project ID not the name. You can find the ID on the project's home page, in the **Project Info** box.

### Authenticate Local Machine

```bash
gcloud auth login --quiet
```

_Note that you can do `gcloud config set project $GCP_PROJECT_ID` and `gcloud config set compute/zone us-central1-a` here and then omit the `--project` and `--zone` options and arguments below._

### Create Jumpbox VM From Local Machine

```bash
gcloud services enable compute.googleapis.com \
  --project "${GCP_PROJECT_ID}"

gcloud compute instances create "jbox-concourse" \
  --image-project "ubuntu-os-cloud" \
  --image-family "ubuntu-1804-lts" \
  --boot-disk-size "200" \
  --machine-type=g1-small \
  --project "${GCP_PROJECT_ID}" \
  --zone "us-central1-a"
```

### SSH To Jumpbox

```bash
gcloud compute ssh ubuntu@jbox-concourse \
  --project "${GCP_PROJECT_ID}" \
  --zone "us-central1-a"
```

### Install Essential Jumpbox Tools

Install tools:

```bash
sudo apt update --yes && \
sudo apt install --yes unzip && \
sudo apt install --yes make && \
sudo apt install --yes ruby
```

```bash
TF_VERSION=0.11.11
wget -O terraform.zip https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip && \
  unzip terraform.zip && \
  sudo mv terraform /usr/local/bin

BOSH_VERSION=5.4.0
wget -O bosh https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-${BOSH_VERSION}-linux-amd64 && \
  chmod +x bosh && \
  sudo mv bosh /usr/local/bin/

BBL_VERSION=7.1.0
wget -O bbl https://github.com/cloudfoundry/bosh-bootloader/releases/download/v${BBL_VERSION}/bbl-v${BBL_VERSION}_linux_x86-64 && \
  chmod +x bbl && \
  sudo mv bbl /usr/local/bin/
```

Verify that these tools were installed:

```bash
which unzip; which make; which ruby; which terraform; which bosh; which bbl
```

### Authenticate Jumpbox

Follow the on-screen prompts as your execute the following:

```bash
gcloud auth login --quiet
```

You must, at this point, log in using the your GCP account credentials in order to perform later adminstrative operations. While this is not good practice in general, this will suffice for testing purposes.

### Setup Jumpbox Variables

In the ubuntu home directory on your jumpbox, create a hidden file named `.env` to store variables, the values of which describe your specific environment. You should customize the `.env` file to suit your target environment before continuing.

Take the template `.env` file below and substitute in the proper values for your GCP project:

```bash
CC_DOMAIN_NAME=CHANGE_ME_DOMAIN_NAME                 # e.g. pivotaledu.io
CC_SUBDOMAIN_NAME=CHANGE_ME_SUBDOMAIN_NAME           # e.g. cls99env66
```

### Persist Your Environment File

Now that we have the `.env` file with our critical variables, we need to ensure that these get set into the shell, both now and every subsequent time the ubuntu user connects to the jumpbox.

```bash
source ~/.env
echo "source ~/.env" >> ~/.bashrc
```

### Increase GCP Quotas

If necessary, increase GCP project quotas (`IAM & admin -> Quotas`) for the __us-central1__ region as follows:

Quota type               | Min Quota Amount
------------------------ | ----------------
CPU                      | > 100
Persistent Disk SSD (GB) | > 1000
In-use IP addresses      | > 32

### BOSH Bootloader (BBL)

[BBL](https://github.com/cloudfoundry/bosh-bootloader) will generate some files, so create a home for this operation and move there:

```bash
mkdir -p ~/bbl-concourse/cloud-config && cd ~/bbl-concourse
```

Create a service account for BBL:

```bash
gcloud iam service-accounts create bbl-service-account \
  --display-name "BBL service account"
  
gcloud iam service-accounts keys create \
  --iam-account="bbl-service-account@$(gcloud config get-value core/project).iam.gserviceaccount.com" \
  bbl-service-account.json
  
gcloud projects add-iam-policy-binding $(gcloud config get-value core/project) \
  --member="serviceAccount:bbl-service-account@$(gcloud config get-value core/project).iam.gserviceaccount.com" \
  --role="roles/editor"
```

Execute BBL to build Jumpbox and BOSH director VM.  
**Note** this also sets the BOSH Director's `cloud config`:

```bash
cat > cloud-config/zz-default.yml << EOF
- type: replace
  path: /vm_types/name=default/cloud_properties/root_disk_size_gb
  value: 200
EOF

bbl up \
  --name concourse \
  --lb-type concourse \
  --iaas gcp \
  --gcp-region us-central1 \
  --gcp-service-account-key ~/bbl-concourse/bbl-service-account.json
```

Extract the external URL and credentials:

```bash
CC_LB_IP=$(bbl lbs | awk -F': ' '{print $2}')
eval "$(bbl print-env)"
```

### Deploy Concourse

Upload a stemcell:

```bash
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-xenial-go_agent
```

Deploy Concourse:

```bash
git clone https://github.com/concourse/concourse-deployment.git ~/bbl-concourse/concourse-deployment/
cd ~/bbl-concourse/concourse-deployment/cluster/
git checkout tags/v4.2.2 # fix at this version

cat >secrets.yml <<EOL
local_user:
  username: admin
  password: adm1npa55w0rd
EOL

bosh deploy -n -d concourse concourse.yml \
  -l ../versions.yml \
  -l secrets.yml \
  --vars-store cluster-creds.yml \
  -o operations/basic-auth.yml \
  -o operations/privileged-http.yml \
  -o operations/privileged-https.yml \
  -o operations/web-network-extension.yml \
  --var network_name=default \
  --var external_url=http://$CC_LB_IP \
  --var web_vm_type=default \
  --var db_vm_type=default \
  --var db_persistent_disk_type=100GB \
  --var worker_vm_type=default \
  --var deployment_name=concourse \
  --var web_network_name=private \
  --var web_network_vm_extension=lb
```

### Inspect the external URL for later use on the local machine

```bash
echo http://$CC_LB_IP
```

### Download `fly` and login

On your *local machine*, navigate to the Concourse web UI (CC_EXTERNAL_URL), download the 
`fly` CLI utils for your OS then log-in via the `fly` CLI:

```bash
fly -t concourse login -c http://$CC_LB_IP
```

Use the username and password added to the `secrets.yml` file created above to login.

### Create a pipeline

Follow [THIS LINK](../shared/README.md) to create your first pipeline using IaaS independent instructions.
