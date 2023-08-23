# k8s-cluster

Deployment instructions for creating a Kubernetes cluster using Rocky Linux 8. See other branches for other OSes.

## Architecture

This will expand as I make the cluster bigger and better:

- The entire cluster will be just **one** node right now
- IP: 10.1.20.81
- Hostname: k01
- OS: Rocky Linux 8

I previously built a cluster with three nodes where two of the nodes were LXC containers and one node was a physical host with Rocky Linux 8 [here](https://github.com/willquill/kube-lxc).

Note: Since I'll probably have more nodes in the future and don't want to rewrite this README, I'll use terms like "all of the nodes" when, obviously, there is only one node right now.

## Install OS

Nothing much to say here. I downloaded the Rocky Linux ISO and installed it onto the destination machines. I just use a 128 GB USB flash drive with a bunch of different ISOs on it and use [Ventoy](https://www.ventoy.net/en/index.html) to choose an ISO from a menu on destination machines.

## Prepare Bastion Host

Presumably, you're going to run your `kubectl` commands from a machine not in the cluster. That's your bastion host. In my case, it's an openSUSE Tumbleweed machine using the Alacritty terminal.

### Prepare SSH Keys and Config

First, you'll want to prepare your SSH keys.

I didn't want to use the default key names of `id_rsa` and `id_rsa.pub` so I ran the following command:

`ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/k8s_rsa`

Since every one of my kubernetes nodes will have a hostname that starts with `k0`, I'm making the SSH config apply to only the `k0*` hosts.

```sh
cat >> ~/.ssh/config<< EOF
Host k0*
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 30
    IdentityFile ~/.ssh/k8s_rsa
EOF
```

Set up the destination for remote authentication. In my case, I'm using the `nova` user on host `k01`.

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub nova@k01`

May as well do root too, just in case:

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@k01`

### Install your terminfo (optional)

Since I'll be SSHing from Alacritty, I'll need the Alacritty terminfo on all machines.

*If you do not install the terminfo, you may sometimes see messages like this:*

```sh
E558: Terminal entry not found in terminfo
'alacritty' not known. Available builtin terminals are:
    builtin_amiga
    builtin_beos-ansi
    builtin_ansi
    builtin_pcansi
    builtin_win32
    builtin_vt320
    builtin_vt52
    builtin_xterm
    builtin_iris-ansi
    builtin_debug
    builtin_dumb
defaulting to 'ansi'
```

First, download the Alacritty repo locally.

`git clone https://github.com/alacritty/alacritty.git`

`cd alacritty`

Now copy `extra/alacritty.info` to remote machines.

`scp extra/alacritty.info nova@k01:/home/nova`

And install the term info on remote machines.

`ssh nova@k01 'sudo tic -xe alacritty,alacritty-direct alacritty.info'`

The next few commands will be run on your destination hosts, where the commands differ based on linux distribution.

### Prepare hosts file

My two hosts:

Host Purpose  |     IP     |       FQDN      | hostname
--------------|------------|-----------------|---------
Bastion       | 10.1.20.40 | lizard.paw.blue | lizard
K8s Node 01   | 10.1.20.81 | k01.paw.blue    | k01

So I want to update the /etc/hosts file on my bastion machine like this:

```sh
sudo tee -a /etc/hosts > /dev/null <<EOT
10.1.20.40 lizard.paw.blue lizard
10.1.20.81 k01.paw.blue k01

# K8s Endpoints
10.1.20.81 k8ep.paw.blue k8ep
EOT
```

Notice that I use `tee` to append because I need root privileges.

### Install Python 3.9

I used [this](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/) guide.

I won't reiterate all the steps from that guide, but here are some of them, as there were some occasional deviations. For some of the steps, I included instructions for both openSUSE and Rocky Linux bastion hosts.

openSUSE only: `sudo zypper in python39`

Rocky Linux only: `sudo dnf -y install python39`

Both OSes:

`sudo pip3 install setuptools-rust wheel`

`sudo pip3 install --upgrade pip`

### Install Ansible

`sudo python3.9 -m pip install ansible`

`ansible --version`

You should see something like this in openSUSE:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/will/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/will/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.9 (main, Nov 17 2021, 09:50:59) [GCC]
  jinja version = 3.0.3
  libyaml = True
```

You should see something like this in Rocky Linux:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/nova/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/nova/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.6 (default, Nov  9 2021, 13:31:27) [GCC 8.5.0 20210514 (Red Hat 8.5.0-3)]
  jinja version = 3.0.3
  libyaml = True
```

## Install Kubernetes

The `k8s-prep.yml` Ansible playbook will do a lot, including installing kubelet, kubeadm, and kubectl.

`ansible-playbook -i hosts -l k8snodes k8s-prep.yml --ask-become-pass`

## Mount NFS

`ansible-playbook -i hosts -l k8sworkers mount-nfs.yml --ask-become-pass`

## Create the cluster

From my bastion host, I'm running a remote SSH command on my node. I couldn't figure out how to do this as my regular user, so I'm doing it as root.

Remember to do this first if you didn't do it earlier:

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@k01`

Do a dry run and then look at your `dryrun.txt` file.

`ssh root@k01 'kubeadm init --dry-run --cri-socket=/var/run/crio/crio.sock --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.23.3' > dryrun.txt`

Do it for real.

`ssh root@k01 'kubeadm init --cri-socket=/var/run/crio/crio.sock --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.23.3'`

If it's successful, you'll see something like this:

```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.20.81:6443 --token <redacted> \
  --discovery-token-ca-cert-hash sha256:<redacted>
```

## Setup Bastion Host

Presumably, you don't want to run your `kubectl` commands on the server all the time.

From your bastion host, install `kubectl` and then do the following with your own node IP to allow your local `kubectl` client to controller the cluster you just created:

```sh
mkdir -p $HOME/.kube &&\
  scp root@k01:/etc/kubernetes/admin.conf $HOME/.kube/config &&\
  chown $(id -u):$(id -g) $HOME/.kube/config
```

## Single Node Cluster (optional)

In a cluster with only one node, you'll need to run this to allow pods to run on the master:

`kubectl taint nodes --all node-role.kubernetes.io/master-`

## Install Pod Network

`kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml`

`kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml`

## Verify Installation

Make sure everything started.

`kubectl get all -A`

If Calico is hanging, you might see some things at 0/1 ready when you run:

`kubectl get all -n calico-system`

You may need to do this on the hosts to fix it:

`systemctl restart crio`

## Install Helm (optional)

I use Helm for a few things, and I didn't have Helm yet on my openSUSE Tumbleweed bastion host, so I did this to install it and get completion working in both bash and ZSH:

`sudo zypper in helm helm-bash-completion helm-zsh-completion helmfile helmfile-bash-completion helmfile-zsh-completion`

## Install Ingress Controller

### NGINX Ingress Controller

See [here](https://kubernetes.github.io/ingress-nginx/deploy/) for documentation.

Install with the following command:

```sh
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Use IPVS kube-proxy

I recommend reading [this doc](https://projectcalico.docs.tigera.io/networking/use-ipvs) to determine if you want to use IPVS instead of iptables.

See guide [here](https://metallb.universe.tf/installation/) for how to enable IPVS kube-proxy.

### Install MetalLB

Installing MetalLB is actually really easy:

`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml`

`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml`

Install the configmap:

`kubectl apply -f manifests/metallb/configmap.yaml`

My configmap looks like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.1.20.210-10.1.20.254
```

The address range is the available address pool from which load balancer services in the cluster will get their IP addresses.

Since my core network uses the `10.1.20.0/24` IP address space, and my DHCP server assigns IPs in the `10.1.20.100-199` range, and because I have static IPs assigned from `1-99` and `200-209`, I'm dedicating `210-254` to be available for cluster load balancers. This means I can deploy 45 load balancers before I've maxed out my address pool.

You can read more about this feature [here](https://blog.inkubate.io/install-and-configure-metallb-as-a-load-balancer-for-kubernetes/), as well as in the official documentation.

### Test MetalLB

The following command will deploy a simple nginx web server with a load balancer service.

`kubectl apply test-nginx.yaml`

You can also apply this other test:

`kubectl apply test-web.yaml`

See your services:

`kubectl get svc -A`

You should see external IPs for your LoadBalancer services, and you should be able to access these from your web browser now!

Here's a sample of what mine look like:

```sh
NAMESPACE          NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
calico-apiserver   calico-api                           ClusterIP      10.105.97.131    <none>        443/TCP                      23h
calico-system      calico-kube-controllers-metrics      ClusterIP      10.96.176.42     <none>        9094/TCP                     23h
calico-system      calico-typha                         ClusterIP      10.103.178.250   <none>        5473/TCP                     23h
default            kubernetes                           ClusterIP      10.96.0.1        <none>        443/TCP                      23h
default            nginx-service                        LoadBalancer   10.98.200.72     10.1.20.211   80:31803/TCP                 6s
ingress-nginx      ingress-nginx-controller             LoadBalancer   10.98.234.83     10.1.20.210   80:31609/TCP,443:32697/TCP   53m
ingress-nginx      ingress-nginx-controller-admission   ClusterIP      10.99.133.230    <none>        443/TCP                      53m
kube-system        kube-dns                             ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP       23h
web                web-server-service                   LoadBalancer   10.108.229.201   10.1.20.212   80:30567/TCP                 3s
```

## Docker Only

If you only want your node(s) to install Docker and nothing else, follow this.

### Populate hosts file

Populate the `hosts` file with the nodes where you want docker only. My only host is `tucana`, so I added this:

```conf
[dockernodes]
tucana ansible_connection=ssh ansible_user=nova
```

Because the `docker-only.yml` file specifies `dockernodes`, you can exclude hosts from the ansible-playbook command.

### Populate local SSH config

Make sure you specify the identify file you want to use

```sh
cat >> ~/.ssh/config<< EOF
Host tucana
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 30
    IdentityFile ~/.ssh/id_rsa
EOF
```

### Set up SSH authentication

Set up the destination for remote authentication. In my case, I'm using the `nova` user on host `tucana`.

`ssh-copy-id -i ~/.ssh/id_rsa.pub nova@tucana`

### Execute playbook

`ansible-playbook -i hosts -l dockernodes docker-only.yml --ask-become-pass`

### Secrets

For passwords and the like, I'm using a Rocky Linux 8 [Linode](https://www.linode.com/) instance as a secrets vault using Hashicorp Vault which I deployed via [these instructions](https://www.linode.com/docs/guides/use-hashicorp-vault-for-secret-management/).

I had to deploy the node, install certbot, acquire a TLS certificate, download Vault files, install Vault, configure Vault

#### Linode Preparation

```sh
sudo dnf install epel-release &&\
  sudo dnf update &&\
  sudo dnf install certbot python3-certbot-nginx nginx -y &&\
  sudo firewall-cmd --zone=public --add-port=80/tcp &&\
  sudo firewall-cmd --zone=public --add-port=443/tcp &&\
  sudo firewall-cmd --reload
```

Double check what you just did for firewall-d:

`firewall-cmd --permanent --zone=public --list-ports`

You should see `443/tcp 80/tcp` as output.

Create the cert:

`sudo certbot --nginx`

You can renew all existing certificates that will expire in under 30 days with the following command:

`certbot renew`

Proceed with TLC Certificate:

`sudo groupadd tls`

```sh
sudo chgrp -R tls /etc/letsencrypt/{archive,live} &&\
  sudo chmod g+rx /etc/letsencrypt/{archive,live} &&\
  sudo find /etc/letsencrypt/archive -name 'privkey*' -exec chmod g+r {} ';'
```

Proceed with vault:

```sh
sudo dnf install unzip -y &&\
curl -O https://releases.hashicorp.com/vault/1.9.3/vault_1.9.3_linux_amd64.zip &&\
  unzip vault_*_linux_amd64.zip &&\
  sudo mv vault /usr/local/bin &&\
  sudo chown root:root /usr/local/bin/vault &&\
  sudo chmod 755 /usr/local/bin/vault &&\
  sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault &&\
  vault --version
```

Sample output: `Vault v1.9.3 (7dbdd57243a0d8d9d9e07cd01eb657369f8e1b8a)`

```sh
sudo useradd --system -d /etc/vault.d -s /bin/nologin vault &&\
  sudo gpasswd -a vault tls &&\
  sudo install -o vault -g vault -m 750 -d /var/lib/vault &&\
  sudo install -o vault -g vault -m 750 -d /etc/vault.d
```

```sh
sudo tee -a /etc/systemd/system/vault.service > /dev/null <<EOT
[Unit]
Description="a tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOT
```

```sh
fqdn=your.fqdn.here
sudo tee -a /etc/vault.d/vault.hcl > /dev/null <<EOT
listener "tcp" {
  address = "0.0.0.0:8200"
  tls_cert_file = "/etc/letsencrypt/live/$fqdn/fullchain.pem"
  tls_key_file = "/etc/letsencrypt/live/$fqdn/privkey.pem"
}

storage "file" {
  path = "/var/lib/vault"
}
EOT
```

`sudo systemctl start vault`

`sudo systemctl enable vault`

Confirm that Vault is operational by using the vault executable to check for the service’s status. Set the VAULT_ADDR environment variable to `https://example.com:8200`, replacing example.com with your own domain:

`export VAULT_ADDR=$fqdn` (you set the fqdn variable a few steps ago)

**This is incomplete, as I left off development with this error:**

```sh
Feb 12 04:30:19 45-56-127-165.ip.linodeusercontent.com systemd[9223]: vault.service: Failed to execute command: Perm>
Feb 12 04:30:19 45-56-127-165.ip.linodeusercontent.com systemd[9223]: vault.service: Failed at step EXEC spawning /u>
-- Subject: Process /usr/local/bin/vault could not be executed
-- Defined-By: systemd
-- Support: https://access.redhat.com/support
--
-- The process /usr/local/bin/vault could not be executed and failed.
```

## Docker Only

If you only want your node(s) to install Docker and nothing else, follow this.

### Populate hosts file

Populate the `hosts` file with the nodes where you want docker only. My only host is `tucana`, so I added this:

```conf
[dockernodes]
tucana ansible_connection=ssh ansible_user=nova
```

Because the `docker-only.yml` file specifies `dockernodes`, you can exclude hosts from the ansible-playbook command.

### Populate local SSH config

Make sure you specify the identify file you want to use

```sh
cat >> ~/.ssh/config<< EOF
Host tucana
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 30
    IdentityFile ~/.ssh/id_rsa
EOF
```

### Set up SSH authentication

Set up the destination for remote authentication. In my case, I'm using the `nova` user on host `tucana`.

`ssh-copy-id -i ~/.ssh/id_rsa.pub nova@tucana`

### Execute playbook

`ansible-playbook -i hosts -l dockernodes docker-only.yml --ask-become-pass`

## License

Distributed under the MIT License. See LICENSE for more information.

## Credit

- [Official documentation for Kubic](https://en.opensuse.org/Kubic:kubeadm)
- [This blog entry by Josphat Mutai](https://blog.sdmoko.net/create-k8s-cluster-with-kubic.html)
