
# Create rancher server with Elemental

```
virt-install -n worldco-rancher --memory 4096 --vcpus 2 --cdrom ~/Downloads/SLE-Micro-5.3-DVD-x86_64-GM-Media1.iso --disk /public/vms/worldco-rancher.qcow2,size=60 --network network=world-co,mac.address=2A:C3:A7
:A7:00:03 --graphics vnc --boot uefi --os-variant slem5.2
```

Install SLE Micro 5.3 by following the steps

Install rke2

```
mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - rancher.world-co.com
EOF
curl -sfL https://get.rke2.io | sh -
echo "export PATH=/var/lib/rancher/rke2/bin:/opt/rke2/bin:\$PATH" >> ~/.bashrc
source ~/.bashrc
systemctl enable --now rke2-server.service
```

Get the kube config to the dev machine.

```
scp rancher.world-co.com:/etc/rancher/rke2/rke2.yaml kubeconfig.yaml
sed 's/127.0.0.1/rancher.world-co.com/' -i kubeconfig.yaml
export KUBECONFIG=$PWD/kubeconfig.yaml
```

Install Cert-manager

From the dev machine:

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true \
    --version v1.10.1
```

Install rancher

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm upgrade --install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --create-namespace \
    --set hostname=rancher.world-co.com \
    --set replicas=1
```

Get the secret for the first authentication:

```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

Install Elemental

In the rancher UI, enable extensions, install the Elemental extension and click the reload button.

Install the Elemental operator from the dev machine using helm:

```
helm upgrade --create-namespace -n cattle-elemental-system --install elemental-operator oci://registry.opensuse.org/isv/rancher/elemental/stable/charts/rancher/elemental-operator-chart
```

# Setup Elemental and prepare the iso

## Sealed Secrets keys

The HTTP certificates and SSH keys will be encrypted in a git repository.
We need to create a key / certificate to encrypt and decrypt those secrets with Sealed Secrets.

Create keys:
```
openssl req -x509 -nodes -newkey rsa:4096 -keyout tls.key -out tls.crt -subj "/CN=sealed-secret/O=sealed-secret"
kubectl -n kube-system create secret tls sealed-secrets-key --cert=tls.crt --key=tls.key --dry-run=client --output yaml >sealed-secrets-key.yaml
```

Edit the generated `sealed-secrets-key.yaml` file and add the following lines in the metadata:

```
  namespace: kube-system
  labels:
    sealedsecrets.bitnami.com/sealed-secrets-key: active
```


## Preparing the registration

To minimize the number of resources to add to add a new store, the following uses the hardware serial number to assign the machines to the store cluster.
The machine registration is common to all stores, and as well as the generated iso.
The only things to change for each store are the machine selector and the cluster definitions.

Apply the `MachineRegistration` configuration:

```
sed "s/SEALED_SECRETS_KEY/`cat sealed-secrets-key.yaml | base64 -w 0`/" src/registration.yaml | kubectl apply -f -
```

Apply the store resources configuration: `kubectl apply -f src/store1234.yaml`

## Generate the iso

Fetch the configuration from the machine registration.

```
wget --no-check-certificate `kubectl get machineregistration -n fleet-default proxy-nodes -o jsonpath="{.status.registrationURL}"` -O initial-registration.yaml
```

Get the script to generate the ISO image:

```
wget -q https://raw.githubusercontent.com/rancher/elemental/main/.github/elemental-iso-add-registration && chmod +x elemental-iso-add-registration
```

Ensure either `podman` is installed or `docker` runs on the dev machine and generate the ISO:

```
./elemental-iso-add-registration initial-registration.yaml
```
Move the ISO file to the /tmp directory for proper access installing virtual machines. 


# Configure Fleet to deploy MetalLB and the Uyuni Proxy

Install `kubeseal` on the dev machine: `zypper in kubeseal`

# Get the SUSE Manager proxy generated config in the Git repository

The Fleet configuration Git repository for this demo is https://github.com/cbosdo/fleet-proxy.
This is matching the configuration for my machines and should be forked and cloned.
The URL of repository should be changed in `src/proxy-repo.yaml` to match the fork.

In Uyuni/SUMA, create a Proxy Configuration, and copy the tar.gz file to the dev instance.
Extract it to a folder with the name of your BRANCH.

For example:

```
mkdir -p ~/store1234
cd ~/store1234
tar zxvf ../store1234-proxy-config.tar.gz
```

You should then have the three branch-specific files needed to generate the secret and config.

```
ls -l ~/store1234
total 28
-rw-r--r-- 1 root root 5925 Feb 10 12:51 config.yaml
-rw------- 1 root root 9330 Feb 10 12:51 httpd.yaml
-rw------- 1 root root 4226 Feb 10 12:51 ssh.yaml
```


Set `GIT_REPO` variable to where the fleet git repository is cloned on the dev machine.
Set 'BRANCH environment variable for the desired BRANCH location.  For example,
```
export GIT_REPO=~/fleet-proxy
export BRANCH=store1234
```
Then run the following command to set the secret and configuration:

```
kubectl create secret generic proxy-secret-import \
    --from-file=httpd.yaml=$BRANCH/httpd.yaml \
    --from-file=ssh.yaml=$BRANCH/ssh.yaml \
    --dry-run client \
    --output json | kubeseal --cert ~/tls.crt -o yaml >${GIT_REPO}/proxy-secrets/overlays/$BRANCH/secrets.yaml
```

Once the secrets are generated, commit and push them in the git repository for Fleet to be able to consume them.

Store the extracted `config.yaml` (one time) in the git repo `proxy/config.yaml` and remove the `proxy_fqdn` line from it.

Commit all changes and push them to the remote git repository:

```
cd ${GIT_REPO}
git commit -a -m "Updated the secrets and config"
git push
```

# Create the proxy machine:


The machines to boot with the Elemental ISO created earlier are required to be UEFI-enabled and have a TPM 2.0 device.
In the demo environment I use virtual machines created as following:
```
virt-install -n worldco-store1234-proxy --memory 4096 --vcpus 2 --cdrom /tmp/elemental-teal.x86_64.iso --disk /public/vms/worldco-store1234-proxy.qcow2,size=60 --network network=world-co,mac.address=2a:c3:a7:a7:00:64 --graphics vnc --tpm emulator,backend.version=2.0 --boot uefi --os-variant slem5.2 --sysinfo system.serial=PXY1234
```
```
virt-install -n worldco-store5678-proxy --memory 4096 --vcpus 2 --cdrom /tmp/elemental-teal.x86_64.iso --disk /public/vms/worldco-store5678-proxy.qcow2,size=60 --network network=world-co,mac.address=2a:c3:a7:a7:00:6E --graphics vnc --tpm emulator,backend.version=2.0 --boot uefi --os-variant slem5.2 --sysinfo system.serial=PXY5678
```

At creation time, SLE Micro 5.3 will be installed on the machines and k3s will be installed on it.   Now launch the Rancher webUI and watch the deployment happen!
