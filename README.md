May 11 2020

Guide to installing TAS4K8s on Microk8s on MacOs (should work with minor tweaks on Linux/Windows)
By Neil Isserow
Customer Success Manager - MAPS
May 2020

This is a guide that I created and worked for me. I would be happy to help anyone wanting to get this working. I am not a TAS/PCF expert, this was just a helpful way for me to use TAS on my Mac with k8s. Please also note versions may be updated below so check for latest! This install takes approx 20 minutes or less on my Mac. I also use nano and other tools, please use whatever you find suitable.

I have been asked why Ubuntu and MicroK8s, the answer is Ubuntu is simple and widely known, and MicroK8s is also simple and avoids having to build using Kubeadm and is well supported.

Please note that I will update and also modify as mistakes etc, are found, please feel free to message me. I hope this is helpful for playing around with TASk8s. I will add a similar for KubeCF and also CFk8s which I have as well soon.

BETA NOTE! You will require access to the Beta to install -  https://tanzu.vmware.com/tas-for-kubernetes 

Release Notes: https://docs.pivotal.io/tas-kubernetes/0-1/release-notes.html

Install Ubuntu 18.04 VM
I simply install on Fusion on my MAC. Take note of the user and pwd you use.

If you use multipass be aware of issues with networking and MetalLB!

Create a new VM I use this for my current Mac spec but you can use larger or maybe smaller (have not tested)
I do a default Ubuntu install from the ISO
Check Microk8s (or install after with sudo snap install microk8s)

ssh to the new Ubuntu Host!

sudo /snap/bin/microk8s.enable dns dashboard metrics-server storage metallb

Choose a IP range when asked for MetalLb that is suitable. I keep it simple and choose the NAT for my VM .200-.240 which is more than enough for testing.

sudo /snap/bin/microk8s.config > kubeconfig

I simply cat this file and copy contents or whatever suits to get it onto your host system

Configure your kubectl config files
For demo I just copy kubeconfig over my current config as I just run 1 at a time but you can append as required for your configuration. I suggest tools like kubens and kubectx if you do this to make it easy to work (brew install kubens kubectx)

Installing TAS4K8S
The official documentation can be found here: https://docs.pivotal.io/tas-kubernetes/0-1/release-notes.html

These were my basic steps:
Download the zip and extract - https://network.pivotal.io/products/tas-for-kubernetes/ 
cd tanzu-application-service….
I first config the registries:
mkdir configuration-values
cd configuration-values
I use docker, you can use whatever is currently supported

nano app-registry-values.yml
(copy below into this file)

#@data/values
---
app_registry:
   hostname: https://index.docker.io/v1/
   repository: "your-login"
   username: "your-login"
   password: "your-pwd"

Nano system-registry-values.yml
(copy below into this file)

#@data/values
---
system_registry:
  hostname: registry.pivotal.io
  username: "your registered login"
  password: "your-pwd"

A quick tip:  the pivotal registry uses harbor and to ensure you can get to it head to registry.pivotal.io and login and you should see all the repositories listed, if not then you are either using the incorrect name or do not have access to the correct repository.

Download all required software as follows:
https://docs.pivotal.io/tas-kubernetes/0-1/installing-command-line-tools.html 

Generate the configuration values:
$ ./bin/generate-values.sh -d "PLACEHOLDER-SYSTEM-DOMAIN" > configuration-values/deployment-values.yml

The PLACEHOLDER is important as it will become the wildcard and be used below for DNS. I stuck to a simple .local for example neilisk8s.local and then everything will prefix for example api.neilsk8s.local etc. etc.

LOAD BALANCER:
As we have configured Metallb as our LB we need to ensure we perform the following prior to setup: mv custom-overlays/replace-loadbalancer-with-clusterip.yaml config-optional this will ensure that the config uses a LB for the service.

INSTALLATION:
You should now be ready to install with the following script provided:

./bin/install-tas.sh configuration-values 

I like to watch so I will execute:

watch kubectl get pods --all-namespaces

Once complete continue to DNS and ensure you know the LB IP!

DNS Setup
You will need DNS for your environment, I use DNSMASQ which is very easy to use. The IP is the address for the public router which you will find once install is completed and should be easy to find via kubectl get svc --all-namespaces and it should be the only 1 listed.

Here are basic instructions:
brew install dnsmasq
cd /usr/local/etc
nano dnsmasq.conf ( you may need to add other items to this file but this is the least)
Add the following to the top (customize as appropriate ie. system.local or mytask8s.local or just .local for all .local):
address=/.local/192.168.64.100
sudo launchctl stop homebrew.mxcl.dnsmasq
sudo launchctl start homebrew.mxcl.dnsmasq
Then test by pinging x.local (or whatever you have set) to ensure all wildcards are pinging the correct address.

Simple CF for testing (yes I am a noob at this and had to write this down :)-)
To test if it worked these are my steps:

cf api --skip-ssl-validation https://api.whateveryouchose.local
cf auth admin PWD-from-configuration-values/deployment-values.yml-file
cf create-org test-org
cf target -o "test-org"
cf create-space dev
cf target -o "test-org" -s "dev"

Finally dont forget to enable the Diego-Docker feature flag

cf enable-feature-flag diego_docker

I have a very simple spring boot to test which is basically this! - https://spring.io/quickstart and follow until you have the app running locally successfully via your browser http://localhost:8080/hello

Once you have this cd to the directory and then:
cf push demo

If you are wondering what will happen here is some info that I have gathered but may need more clarity as well as definition as this is all I can see:
You can watch on your hub.docker.com repositories and a new repository should be created with a GUID (this will be public repository bear that in mind! You can change this to private after however the GUID will change each time and for the free docker hub you only get 1 private repository before having to pay so might want to switch to a local harbor registry)
TAS will do this deploy in steps so could take a while, I use k9s to watch but you will see the following pods/containers that are helpful:
In the first instance you will see the namespace cf-workloads-staging create a guid which you can follow, basically a number of steps as follows (not sure of exact order) - analyze, detect,  build, prepare, restore, export, completion.

Next the namespeace  cf-system creates a pod with guid and containers as follows: istio-init, istio-proxy and cfroutesync. These seem self explanatory and are creating the route/sidecar etc.

Another pod created in cf-system NS for the eirini-guid similar to above

Another pod created in cf-system NS for logs

Finally in the NS cf-workloads a pod with guid demo-guid created


There may be a few more I have missed but if you get this far and have this running with this simple demo spring boot app you should be able to just head to the route in your browser for example: http://demo.myk8s.local/hello and voila it’s working.

Service Broker and UI (work in progress from here...)

Lets add a service broker and a simple OS UI to our system. We will add Stratos as the UI and Minibroker as the service broker.

Stratos UI

https://github.com/cloudfoundry/stratos/tree/master/deploy/cloud-foundry

There are 3 ways to deploy Stratos, using docker (does not persist), using a Helm chart or even as a CF application. The one you choose is up to you. I have tried all three and had mixed results based on my setup. Docker is super simple tho and considering the small amount of persistence, which is really just the connection to your TAS system there are few downsides for a demo system.

Docker Instructions:

docker run -p 4443:443 splatform/stratos:latest

CF Instructions:

Download https://github.com/cloudfoundry/stratos/blob/master/manifest-docker.yml 
nano manifest-docker.yml
Add the following:
services:
- console_db

echo > redis.json '[{ "protocol": "tcp", "destination": "192.168.113.0/24", "ports": "5432", "description": "Allow Postgress traffic" }]'
cf create-security-group postgress_networking redis.json
cf bind-security-group   redis_networking org space



cf create-service postgresql v9.4 console_db

cf push -f manifest-docker.yml

Minibroker

-----
