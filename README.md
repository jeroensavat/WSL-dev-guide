# Enhancing Your Windows Development Environment for AWS/k8s/terraform with WSL

Note this guide is opinionated, and assumes you are using Windows 10 or later with WSL2.

## Installing Windows Subsystem for Linux (WSL)
Choose Debian as your preferred distribution for WSL. 
This provides a stable and widely-supported Linux environment.

In an administrative PowerShell prompt, run the following commands:

```powershell
# install wsl if it isn't already
wsl --install
# list available distros
wsl --list --online
# opinionated: use Debian
wsl --install -d Debian
```

## Installing Kubernetes Command-Line Tool (kubectl)
Kubernetes is essential for managing containerized applications, and kubectl is its command-line interface.

```bash
# Downloading kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# Verifying the Download
# Ensure the integrity of kubectl by verifying its checksum.
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
# Installing kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
# Cleanup and Verification
# Remove the downloaded files and check if kubectl works correctly.
rm kubectl && rm kubectl.sha256
kubectl version --client
```

## Installing AWS Command Line Interface (CLI)
Needed for managing AWS resources directly through the command line.

```bash
# Downloading AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
# Installing AWS CLI
sudo ./aws/install
# Cleanup
rm awscliv2.zip && rm -rf aws
```

## Installing aws-vault
aws-vault securely stores and accesses AWS credentials.

```bash
curl -LO https://github.com/99designs/aws-vault/releases/download/v7.2.0/aws-vault-linux-amd64
chmod +x aws-vault-linux-amd64
sudo mv aws-vault-linux-amd64 /usr/local/bin/aws-vault
```

## Setting Up Pass (Default Key Vault for AWS Credentials)
pass is a simple password manager that integrates with aws-vault.

Installation and Key Generation:
```bash
sudo apt install pass
gpg --generate-key
# Follow the prompts to set up your GPG key.

# Initialize and Trust the GPG Key
# Use 'gpg --list-keys' to find your GPG Key ID.
pass init -p aws-vault [Your GPG Key ID]
# trust your key
gpg --edit-key [Your GPG Key ID], trust, 5, save
```

Set the Default Backend for aws-vault
```bash
echo "export AWS_VAULT_BACKEND=pass" >> ~/.bashrc
source ~/.bashrc
```

## Installing 1Password CLI (optional)
Manage your 1Password vaults and items from the command line.
(opinionated): add mfa keys to 1pass for your aws account(s) with otp.
https://support.1password.com/one-time-passwords/

```bash
# download
curl -LO https://cache.agilebits.com/dist/1P/op2/pkg/v2.23.0/op_linux_amd64_v2.23.0.zip
# unzip
unzip op_linux_amd64_v2.23.0.zip
# make executable
chmod +x op
# cleanup
rm op.sig
# move to bin
sudo mv op /usr/local/bin/op
```

## Configuring Symbolic Links for Configuration Files
Link your Windows configuration files for .kube and .aws to WSL.
(Also add your ssh keys so you can add them later)

```bash
ln -s /mnt/c/Users/[YourWindowsUsername]/.kube ~/.kube
ln -s /mnt/c/Users/[YourWindowsUsername]/.aws ~/.aws
ln -s /mnt/c/Users/[YourWindowsUsername]/.ssh ~/.ssh
```

## Adding AWS Profiles with aws-vault
Manage multiple AWS profiles securely.

First an example of an .aws/config file where "i" is the main profile - recommended that your setup looks like this.
Other profiles are IAM roles that can be assumed from the main profile.
Sandbox is a separate AWS account with no IAM roles, "standard" access.
note: you only need an .aws/config, no .aws/credentials file thanks to aws-vault!

<1pass-account-name> is the name of the 1pass account where you store your mfa keys for your aws account(s).
```op account list``` will give you a list of all your 1pass accounts. the name is the first entry.
<aws-acount-id> is the id of your aws account. you can find it in the aws console, top right corner.
<your.user> should be the id of your MFA device. make sure you use your name.surname as the ID when setting up.

```text
[profile i]
mfa_serial = arn:aws:iam::<aws-acount-id>:mfa/<your.user>
mfa_process=op --account <1pass-account-name> item get "LS - AWS - PROD" --otp
region = eu-west-1

[profile eks-lec-staging-readonly]
source_profile = i
role_arn = arn:aws:iam::<aws-acount-id>:role/eks-lec-staging-readonly
mfa_serial = arn:aws:iam::<aws-acount-id>:mfa/<your.user>
mfa_process=op --account <1pass-account-name> item get "LS - AWS - STAG" --otp
region = eu-west-1

[profile eks-lec-production-readonly]
source_profile = i
role_arn = arn:aws:iam::<aws-acount-id>:role/eks-lec-production-readonly
mfa_serial = arn:aws:iam::<aws-acount-id>:mfa/<your.user>
mfa_process=op --account <1pass-account-name> item get "LS - AWS - PROD" --otp
region = eu-west-1

[profile sandbox-admin]
region = eu-west-1
```

```bash
# should give you a list of all your profiles:
aws-vault ls
# add your main profile
aws-vault add i
# this will prompt you to enter your aws key and secret.
# you set this up in your AWS console under "security credentials" - Access Keys.
# there should be only one per account
```

## Setting Up Kubernetes Profiles
Configure Kubernetes contexts for different environments.

```bash
aws-vault exec [aws-profile]
# needed only once ever - this sets up your .kube config so you can connect to a specific k8s cluster
# (eg. lec-staging, lec-production or default-sandbox)
aws eks update-kubeconfig --name [cluster-name] 
# check if it worked:
kubectl get pods -n [namespace]
```

## Example usage of 1pass + aws-vault

```bash
# will ask for your 1pass password
eval $(op signin)
# exec into an AWS profile, which 
aws-vault exec [aws-profile]
```

### pro-tip: log in to 1pass at login
Add a 1pass login prompt to your .bashrc.
This will prompt you for your 1pass password every time you open a new terminal.

```bash
echo "eval $(op signin)" >> ~/.bashrc
```

## k9s (Visual Interface for kubectl)
Enhance your Kubernetes command-line experience with a visual interface.

```bash
curl -LO https://github.com/derailed/k9s/releases/download/v0.28.2/k9s_Linux_amd64.tar.gz
tar -xzvf k9s_Linux_amd64.tar.gz
chmod +x k9s
sudo mv k9s /usr/local/bin/k9s
k9s
```

## Configuring Git for WSL
Set up Git to work seamlessly between Windows and WSL.

```bash
git config --global core.autocrlf true
git config --global core.sshcommand "ssh -i /home/<user>/.ssh/id_rsa"
# Ensure SSH agent is running and add your SSH key.
eval `ssh-agent -s`
ssh-add /home/<user>/.ssh/id_rsa
```

## (optional) Enhanced cd Command for Windows Paths 
Modify the cd command in .bashrc to handle Windows paths.
(add this  to the end of your .bashrc)

```bash
cd() {
    if [[ "$1" =~ ^[a-zA-Z]: ]]; then
        # It's a Windows path
        local win_path="${*//\\//}"  # Replace backslashes with forward slashes in all arguments
        local drive="${win_path:0:1}"  # Extract drive letter
        local path="${win_path:2}"  # Extract path after drive letter
        local wsl_path="/mnt/${drive,,}/${path//:/}"  # Convert drive letter to lowercase, remove colon, and construct WSL path
        command cd "$wsl_path"
    else
        # It's a Unix path
        command cd "$@"
    fi
}
```

### (optional) Installing various common tools/utils
Not installed by default on debian but (in my experience) needed sooner or later:
python3, jq, curl

```bash
sudo apt update && sudo apt upgrade
sudo apt install python3 python3-pip ipython3
sudo apt install jq
sudo apt install curl
```

## TODO: Add instructions for setting up Docker for WSL
## TODO: Add instructions for setting up terraform tools (terraform, tfenv, tflint)
