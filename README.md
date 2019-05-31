# Cloud Foundry + openSUSE Stemcell
## Pre-Require
### docker
```
curl -fsSL https://get.docker.com/ | sudo sh
sudo usermod -aG docker $(whoami)
sudo reboot
```
### rvm
```
gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
\curl -sSL https://get.rvm.io | bash
source ~/.rvm/scripts/rvmw
rvm install ruby-2.3.1
rvm --default use 2.3.1
```
### bosh-cli
```
wget https://github.com/cloudfoundry/bosh-cli/releases/download/v5.5.1/bosh-cli-5.5.1-linux-amd64
chmod +x bosh-cli-*
sudo mv bosh-cli-* /usr/local/bin/bosh
bosh -v
```
### cf-cli
```
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
sudo apt-get update
sudo apt-get install cf-cli
```
### go-lang
```
wget https://dl.google.com/go/go1.12.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.12.5.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.bashrc
echo 'export PATH=$PATH:$GOROOT/bin' >> $HOME/.bashrc
. .bashrc
go version
echo $GOROOT
```
### credhub
```
wget https://github.com/cloudfoundry-incubator/credhub-cli/releases/download/2.4.0/credhub-linux-2.4.0.tgz
tar -xvf credhub-linux-2.4.0.tgz
sudo mv credhub /usr/local/bin

```

## Build image and run Docker container
### Building image
```
cd ~/workspace
git clone https://gitlab.com/abhisr/suse-openstack-stemcell.git
cd suse-openstack-stemcell/ci/docker/suse-os-image-stemcell-builder
wget https://github.com/richardatlateralblast/ottar/blob/master/VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle?raw=true
mv VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle* VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle
./build
```
> If there is a problem with the build in the OpenStack Inception, build on the local PC and upload the tgz file to the OpenStack.
```
cd ~/workspace/suse-openstack-stemcell/ci/docker/
chmod 755 run
./run suse-os-image-stemcell-builder
# Enter the Docker container
```
### Building openSUSE Openstack Stemcell
Inside the Docker container
```
export SKIP_UID_CHECK=1
mkdir -p /opt/bosh/tmp
bundle install
bundle exec rake stemcell:build_os_image[opensuse,leap,/opt/bosh/tmp/os_leap_base_image.tgz]
# It takes about 40 minutes....
```
```
export BOSH_MICRO_ENABLED=no
bundle exec rake stemcell:build_with_local_os_image[openstack,kvm,opensuse,leap,/opt/bosh/tmp/os_leap_base_image.tgz]
exit
```
### Copy openSUSE Openstack Stemcell
Copy openSUSE Openstack Stemcell from container to your localhost (NoteBook)
```
# docker cp YOUR-CONTAINER-ID:/opt/bosh/tmp/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz YOUR-LOCAL-PATH/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz
docker cp $(docker ps -a -q --filter ancestor=bosh/suse-os-image-stemcell-builder):/opt/bosh/tmp/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz \
   $HOME/workspace/suse-openstack-stemcell/tmp/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz
```
> Copy Openstack Stemcell Raw format, only if your Openstack use Ceph Storage instead of cinder
```
# docker cp YOUR-CONTAINER-ID:/opt/bosh/tmp/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent-raw.tgz YOUR-LOCAL-PATH/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent-raw.tgz
```

> upload the tgz file to the OpenStack.
```
# scp -i /home/hyo/ssh_key/opensuse.pem bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz ubuntu@192.168.40.117:/home/ubuntu/
```

>https://bosh.io/docs/build-stemcell/
>https://github.com/cloudfoundry/bosh-linux-stemcell-builder/blob/master/README.md
>https://github.com/cloudfoundry/bosh-linux-stemcell-builder/blob/master/ci/docker/suse-os-image-stemcell-builder/README.md

## Deploying Micro-Bosh on openSUSE Stemcell
### Get the bosh-deployment 
```
cd workspace
git clone https://github.com/cloudfoundry/bosh-deployment.git
cd bosh-deployment
```
### change bosh release ubuntu-xenial to uncompiled release
#### bosh.yml
##### before
```
releases:
- name: bosh
  sha1: ac6c0199c5811c5514791f2256a880e9514c565d
  url: https://s3.amazonaws.com/bosh-compiled-release-tarballs/bosh-270.0.0-ubuntu-xenial-315.26-20190520-210417-660417018-20190520210427.tgz
  version: 270.0.0
- name: bpm
  sha1: 3ed31e60462164509df7fcd4d5791fa14e03cda2
  url: https://s3.amazonaws.com/bosh-compiled-release-tarballs/bpm-1.0.4-ubuntu-xenial-315.26-20190517-195714-698683942-20190517195721.tgz
  version: 1.0.4
```
##### after
```
releases:
- name: bosh
  sha1: 0ec22e86a1537e2af5316273140b0060a3504134
  url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=270.0.0
  version: 270.0.0
- name: bpm
  sha1: c2cceb2d1e271a2f7c5e7c563a7b26f919ebc17a
  url: https://bosh.io/d/github.com/cloudfoundry/bpm-release?v=1.0.4
  version: 1.0.4
```
#### uaa.yml
##### before
```
    name: uaa
    sha1: 1453727887979fbf1118f57b1a07e0d287d30b69
    url: https://s3.amazonaws.com/bosh-compiled-release-tarballs/uaa-64.0-ubuntu-xenial-315.34-20190523-225044-859113139-20190523225104.tgz
    version: "64.0"
```
##### after
```
    name: uaa
    sha1: f10caefdd136675975bfc88d6409f322c68209d4
    url: https://bosh.io/d/github.com/cloudfoundry/uaa-release?v=64.0
    version: "64.0"
```
#### credhub.yml
##### before
```
    name: credhub
    sha1: 652fa3b46f67551990d36ed1a8a19af22d41c200
    url: https://s3.amazonaws.com/bosh-compiled-release-tarballs/credhub-2.4.0-ubuntu-xenial-315.34-20190523-223859-194559593-20190523223904.tgz
    version: 2.4.0
```
##### after
```
    name: credhub
    sha1: 2e08e5de86288f421fb7eff72a095adb78c31ea8
    url: https://bosh.io/d/github.com/pivotal-cf/credhub-release?v=2.4.0
    version: 2.4.0
```

### change bosh release ubuntu-xenial to uploaded release tgz file
#### openstack/cpi.yml
##### before
```
- name: stemcell
  path: /resource_pools/name=vms/stemcell?
  type: replace
  value:
    sha1: 20a113058c97fa8fa445ae60c6b6217ebcf91440
    url: https://s3.amazonaws.com/bosh-core-stemcells/315.34/bosh-stemcell-315.34-openstack-kvm-ubuntu-xenial-go_agent.tgz
```
##### after
```
- name: stemcell
  path: /resource_pools/name=vms/stemcell?
  type: replace
  value:
    #sha1: 20a113058c97fa8fa445ae60c6b6217ebcf91440
    url: file:///home/ubuntu/bosh-stemcell-0000-openstack-kvm-opensuse-leap-go_agent.tgz
```

### deploy Micro-Bosh
```
bosh create-env bosh.yml \
  --state=../deployments/opensuse/state.json \
  --vars-store=../deployments/opensuse/creds.yml \
  -o openstack/cpi.yml \
  -o jumpbox-user.yml \
  -o uaa.yml \
  -o credhub.yml \
  -v director_name=opensuse \
  -v internal_cidr=10.0.200.0/24 \
  -v internal_gw=10.0.200.1 \
  -v internal_ip=10.0.200.10 \
  -v auth_url=http://192.168.40.15:5000/v3/ \
  -v az=nova \
  -v default_key_name=opensuse \
  -v default_security_groups=[bosh] \
  -v net_id=a7e2912a-9c29-412b-8879-76704d3ad74d \
  -v openstack_username=openpaas \
  -v openstack_password=crossent1234 \
  -v openstack_domain=default \
  -v openstack_project=openpaas \
  -v private_key=~/.ssh/opensuse.pem \
  -v region=RegionOne
```

### check the Micro-Bosh VM filesystem
```
bosh int ~/workspace/deployments/opensuse/creds.yml --path /jumpbox_ssh/private_key > ~/.ssh/jumpbox.key
chmod 600 ~/.ssh/jumpbox.key
ssh -i ~/.ssh/jumpbox.key jumpbox@10.0.200.10
```
```
bosh/0:~$ uname -r
4.4.179-99-default
bosh/0:~$ uname -a
Linux 1417798b-16b4-4be8-7eca-f29fd9ef9392 4.4.179-99-default #1 SMP Tue May 14 18:07:16 UTC 2019 (c775d39) x86_64 x86_64 x86_64 GNU/Linux
bosh/0:~$ cat /etc/SuSE-release 
openSUSE 42.3 (x86_64)
VERSION = 42.3
CODENAME = Malachite
# /etc/SuSE-release is deprecated and will be removed in the future, use /etc/os-release instead.
```

## Deploying CF on Ubuntu Stemcell
### Alias and log into the Director
```
bosh alias-env paxpt -e 10.0.200.10 --ca-cert <(bosh int ~/workspace/deployments/opensuse/creds.yml --path /director_ssl/ca)
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int ~/workspace/deployments/opensuse/creds.yml --path /admin_password`
```
### Update cloud config
```
git clone https://github.com/cloudfoundry/cf-deployment ~/workspace/cf-deployment
cd ~/workspace/cf-deployment
bosh -e paxpt update-cloud-config iaas-support/openstack/openstack-cloud-config.yml
```
#### Upload a runtime-config
```
bosh -e vbox update-runtime-config ~/workspace/bosh-deployment/runtime-configs/dns.yml --name dns
```
### Upload stemcell
```
bosh -e paxpt upload-stemcell --sha1 d79a7bc1df3152941e0fa16345a6088c1d1f372e https://bosh.io/d/stemcells/bosh-openstack-kvm-ubuntu-xenial-go_agent?v=250.17
```
### Deploy CF
```
bosh -e paxpt -d cf deploy cf-deployment.yml \
    -o operations/openstack.yml \
    -o operations/use-haproxy.yml \
    -o operations/use-haproxy-public-network.yml \
    -o operations/scale-to-one-az.yml \
    -v haproxy_public_network_name=floating \
    -v haproxy_public_ip=192.168.40.105 \
    -v system_domain=192.168.40.105.xip.io
```
### Test
#### Set CredHub
```
export BOSH_CA_CERT="$(bosh interpolate ~/workspace/deployments/opensuse/creds.yml --path /director_ssl/ca)"

export CREDHUB_SERVER=https://10.0.200.10:8844
export CREDHUB_CLIENT=credhub-admin
export CREDHUB_SECRET=$(bosh interpolate ~/workspace/deployments/opensuse/creds.yml --path=/credhub_admin_client_secret)
export CREDHUB_CA_CERT="$(bosh interpolate ~/workspace/deployments/opensuse/creds.yml --path=/credhub_tls/ca )"$'\n'"$( bosh interpolate ~/workspace/deployments/opensuse/creds.yml --path=/uaa_ssl/ca)"
```
Download release file
> https://github.com/cloudfoundry-incubator/credhub-cli/releases
and install
```
tar -xvf credhub-linux-2.2.0.tgz
mv credhub /usr/local/bin
```
and retrieve credentials in CredHub
```
credhub login -s https://192.168.50.6:8844 --skip-tls-validation
credhub find
credhub find -n cf_admin_password
credhub get -n <FULL_CREDENTIAL_NAME>
credhub get -n /bosh-lite/cf/cf_admin_password
```
#### App Push
```
cf login -a https://api.192.168.40.105.xip.io --skip-ssl-validation
cf create-space manager -o system
cf target -o system
```
```
vi index.php

<?php
  phpinfo();
?>
```
```
cf push testphpapp
```
