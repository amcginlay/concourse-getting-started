# concourse-getting-started
### # What's the absolute minimum I have to do to get Concourse running on VirtualBox?

Download VirtualBox and check version (min 5.1)

```
brew cask install virtualbox
vboxmanage --version
```

Download bosh CLI v2 and check version (min 2.0)

```
brew install cloudfoundry/tap/bosh-cli
bosh2 --version
```

Grab the offical deployment files and create a new folder for our deployments

```
REPO_BOSH=cloudfoundry/bosh-deployment
git clone https://github.com/$REPO_BOSH ~/src/$REPO_BOSH/
cd ~/src/$REPO_BOSH/ && mkdir ./deployments/
```

Create the BOSH Lite Director VM (this may take a few minutes)

```
# assuming working directory is ~/src/$REPO_BOSH/
bosh2 create-env ./bosh.yml \
  --state ./deployments/state.json \
  -o ./virtualbox/cpi.yml \
  -o ./virtualbox/outbound-network.yml \
  -o ./bosh-lite.yml \
  -o ./bosh-lite-runc.yml \
  -o ./jumpbox-user.yml \
  --vars-store ./deployments/creds.yml \
  -v director_name="Bosh Lite Director" \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork
```

Set up an alias named `vbox` for the BOSH Lite Director VM and confirm it worked

```
bosh2 -e 192.168.50.6 --ca-cert <(bosh2 int ~/src/$REPO_BOSH/deployments/creds.yml --path /director_ssl/ca) alias-env vbox
bosh2 environments # NOTE the '192' IP Address
curl -k https://192.168.50.6:25555/info
```

Configure environment variables for your login credentials, comparing status before and after (remember if you move to a new terminal window you'll need to recreate these)

```
bosh2 -e vbox environment
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh2 int ~/src/$REPO_BOSH/deployments/creds.yml --path /admin_password`
bosh2 -e vbox environment
```

Upload a suitable stemcell and check it

```
bosh2 -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent?v=3421.11
bosh2 -e vbox stemcells
```

Upload `Concourse` and `Garden-runC` releases then check them

```
bosh2 -e vbox upload-release https://bosh.io/d/github.com/concourse/concourse?v=3.3.2
bosh2 -e vbox upload-release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release?v=1.9.0
bosh2 -e vbox releases
```

Grab the `concourse-getting-started` repo.

```
REPO_STARTER=amcginlay/concourse-getting-started
git clone https://github.com/$REPO_STARTER ~/src/$REPO_STARTER/
```

Update the cloud configuration and check it.  This contains the IAAS specific information.

```
bosh2 -e vbox update-cloud-config ~/src/$REPO_STARTER/cloud-config.yml
bosh2 -e vbox cloud-config
```

Deploy `Concourse` and check it (this may take a few minutes)

```
bosh2 -e vbox -d concourse deploy ~/src/$REPO_STARTER/concourse.yml
bosh2 -e vbox deployments
```

Ensure Concourse is reachable from a browser ...

```
open http://10.244.8.2:8080/
```

... if not you should open up the '10.244' routes for BOSH Lite and retry.  NOTE the '192' IP is the one we allocated for the BOSH Lite Director VM earlier.

```
sudo route delete -net 10.244.0.0/16
sudo route add -net 10.244.0.0/16 192.168.50.6
netstat -rn | grep 10.244 # ensure route set correctly
```

Follow the browser prompts to download the Concourse CLI tool named `fly` then install it on the path and check it.

```
install ~/Downloads/fly /usr/local/bin
fly --version
```

Now login to Concourse and check

```
fly -t concourse login --concourse-url http://10.244.8.2:8080
fly targets
```

Follow the warning prompts to re-sync `fly` if necessary

```
fly -t concourse sync
```

Now follow the [IaaS independent instructions](../shared/README.md) to create your first pipeline.

### Task Complete!
