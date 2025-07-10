---
title: "Setting Up Our Jumphost and Dev Box"
summary:  "Creating our jumphost box for mangeining the rest of our infrastrucutre and use for basic development for our images"
date: 2025-07-09
tags: ["deployment", "jumphost"]

---
First, we’re going to set up the dev box — a place to store and push our infrastructure code to Git, and a console to work from. This box should most likely be restricted to just you, the administrator.

Let’s look at the tools we’ll need on this box:

- **git** — to push and manage our code changes
    
- **ansible** — to test our Infrastructure as Code (IaC) and deploy our dev environment
    
- **podman** — can be run as a user to build and push our test images without needing root access
    
- **kubectl** — once our testing/staging cluster is up, we’ll use this to access it
    
- **kustomize** — for troubleshooting our `kustomize` deployments
    
- **helm** — to push newly developed charts to a registry and test them
    
- **sops** — needed for encrypting secrets if we use Flux
    
- **age** — used to generate secure keys for encrypting with SOPS
    
- **flux** — enables GitOps for the dev cluster
    
- **go** — required to install `k0sctl`
    
- **k0sctl** — used to deploy `k0s`
    
- **cilium CLI** — needed for testing Cilium and checking its status
    

---

First, we’re going to create some SSH keys. We’ll replace the login key later when we set up an SSH CA, but for now we need one to build the environment.

Generate your login key (my account is `admin1`):

```css

ssh-keygen -t ed25519 -f ~/.ssh/admin1_ed25519_key -C "administration key"

```

Now let’s create the authorized keys file:

```css
cat ~/.ssh/admin1_ed25519_key.pub >> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys
```

This allows you to log in with your SSH key instead of a password:

```css
passwd -d admin1
```

My default image is based on Ubuntu’s cloud image with `cloud-init`, which automatically adds the `admin1` account to the sudoers file with `NOPASSWD`. Still, I like to ensure there’s no password set for the account.

Most of your images will mimic the cloud VPS. Before disabling the password, make sure your `/etc/sudoers` looks something like this:

```css
# User rules for admin1
admin1 ALL=(ALL) NOPASSWD:ALL
```

Keep in mind this weakens some of your machine’s security but is needed for deployments.

To disable the password:

```css
sudo passwd -d admin1
```

Generate your Git key:

```css
ssh-keygen -t ed25519 -f ~/.ssh/admin1_git_ed25519_key -C "administration git key"
```

And finally, generate a Flux key:

```css
ssh-keygen -t ed25519 -f ~/.ssh/admin1_flux_ed25519_key -C "flux sync git key"
```

Once Flux is installed, consider moving this key to a more secure location.

---

### Install Tools

Start with what we can get from `apt`:

```css
sudo apt install git podman golang-go python3-pip pipx age -y
```

#### Fix Podman Registry Error

Podman might complain about missing registries. To fix this:

```css
mkdir -p ~/.config/containers/
echo 'unqualified-search-registries = ["docker.io"]' >> ~/.config/containers/registries.conf
```

#### Install podman-compose

```css
pipx install podman-compose
pipx ensurepath
```

#### Install Ansible

```css
pipx install --include-deps ansible
```

#### Install k0sctl

```css
apt-get install golang-go

go install github.com/k0sproject/k0sctl@latest
cd ~/go/bin
sudo mv k0sctl /usr/local/bin
k0sctl completion > k0sctl
sudo mv k0sctl /etc/bash_completion.d/k0sctl
```

#### Install Helm

```css
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

#### Install Cilium CLI

```css
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

#### Install Kustomize

```css
cd ~/.local/bin
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
```

#### Install Flux

```css
curl -s https://fluxcd.io/install.sh | sudo bash
```

#### Install SOPS

```css
curl -LO https://github.com/getsops/sops/releases/download/v3.10.2/sops-v3.10.2.linux.amd64
sudo mv sops-v3.10.2.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
```

---

 
    

In my next few blogs,  I plan to set up Gitea to store our local code. We want a local Git repo so we don’t leak things like domain names or other internal info to the blue team.

Also I plan to make a crash course in ansible to create some basic ansible scripts:


- patching and reboot
    
- drive expansion script
    
- deploying docker

- a deployment script for setting up this box

---
At this point, the dev box should have all the tools needed to start building out the environment.

