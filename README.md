# Ansible for Unix-Like Artifacts Collector (UAC)
* This Ansible playbook helps to run the UAC in an easy-to-use, repeatable way, that minimizes human error and can be run across mutliple servers at the same time.
* The amount of pre-work does not make this all that useful for one-off data collection, but could be helpful when used on multiple servers, or if you have to do this process more than once.
* Once the pre-work is done, onboarding new uac targets and running the playbook should be relatively easy as compared to doing so multiple times.

## Pre-Requisites
* SSH access to the remote server[s]
* Python installed on the remote server[s] (this is standard)
* Targeting Unix-like (i.e. Linux) remote server[s]
* ansible 2.9 or later installed on your server or local workstation (i.e. Macbook or Windows Subsystem for Linux) ([link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pipx-install))
* python3 installed on your server or local workstation ([link](https://www.python.org/downloads/))
* git installed on your server or local workstation ([link](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git))
* An SSH key that you can use to access the remote server[s] with via SSH
* The uac tool tar.gz file downloaded to your workstation

## 1. Git clone this repository
* From a terminal (recommend using the integrated terminal in VS Code with the Ansible extension installed), clone the repository in a folder you'd like to store the project in:
```
git clone <repo address>
```
* Change into the cloned repository:
```
cd uac-ansible
```
## 2. Copy over the uac tool
* Copy the uac tool's .tar.gz file in the `files/uac/<uac-version>` folder corresponding to your uac tool's version.
* i.e `files/uac/2.9.1/uac-2.9.1.tar.gz`

## 3. Set target servers in inventory
* Open inventory.yaml in a text editor or IDE (VS Code with Ansible extension recommended)
* Replace the example IP addresses (1.1.1.1 and 1.1.1.2) with a list of the public IPv4 addresses of the servers you'd like to target with Ansible and the uac tool. 
* Ensure that you retain the correct indentation and put a colon(:) at the end of each entry as shown below:
```
---
all:
  hosts:
    localhost:
  children:
    uac_target:
      hosts:
        1.1.1.1:
        1.1.1.2:
```
## 4. Create host_vars files 
* In the [host_vars](host_vars) folder, create an file for each of the uac_target servers listed in the inventory.yaml file from the previous step. 
* The name of the file must match what was in the inventory.yaml file, and add a `.yaml` extension, i.e. `host_vars/1.1.1.1.yaml`
* Then, replace the `ansible_user` and `ansible_ssh_private_key_file` variables to match the server's specifications.
* Leave the `ansible_become_password` blank for now, that will be done in the next step.
```
---
ansible_user: example-user
ansible_ssh_private_key_file: ~/.ssh/example-server.pem
ansible_become_password: 
```
## 5. Encrypt the `ansible_become_password` variable
* Create a file called `.password.txt` in your project's root folder.
* Type in a password for your Ansible Vault.
* Change the permissions of the `.password.txt` file to be readable only by you:
```
chmod 600 .password.txt
```
* In a terminal, type in the following command and hit `Enter`:
```
ansible-vault encrypt_string
```
* Copy or type in the server's sudo password
* Hit `ctrl+d` twice. DO NOT HIT ENTER. After typing it out, just hold the `control` key and hit the `D` key two times.
* Copy the output all the way from `!vault |` to the end of the block of numbers.
* Paste that into the server's host_vars file, completely replacing the previous value for `ansible_become_password`, i.e:
```
...
ansible_become_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          39663636393132396438633330396261636566616161346333623766613734373235363862396430
          6434663833623464306334366336653462636637643036310a653632633539373865373463313834
          30633836393666386634343364636139646530663837383563663337396133333061356161336637
          3033333430363465640a306339363231323037323562613866616330643761633161383236323663
          3165
```
## 6. Run the uac.yaml playbook
* This playbook will target all of the servers in the uac_target list in your inventory.yaml, and will use the variables in each of their host_vars files to run the uac tool and extract the results.
* The results will be stored at files/results and the name of the folder that the resulting .tar.gz and .log files will be stored in for each host will be named after the IP address and the timestamp.
```
ansible-playbook uac.yaml
```
* Extended playbook explanation at the bottom of this file.

## Additional Notes:
### Troubleshooting
* For increased verbosity, or for debugging purposes, add extra v's to the command, i.e:
```
ansible-playbook uac.yaml -vvv
```
### Contact:
* This Ansible playbook was made available as-is and should not be expected to be supported.
* If you do have questions, please contact Jacob Emery (jacob.emery@ibm.com)

### Playbook explanation
#### Summary: 
This playbook is used to run the Unix-like Artifacts Collector (UAC) tool on a set of target servers. UAC is a Live Response collection script for Incident Response that makes use of native binaries and tools to automate the collection of AIX, ESXi, FreeBSD, Linux, macOS, NetBSD, NetScaler, OpenBSD and Solaris systems artifacts. It was created to facilitate and speed up data collection, and depend less on remote support during incident response engagements. ([link](https://tclahr.github.io/uac-docs/)). The playbook uses Ansible's built-in modules to copy over the UAC tool from a source directory, run it, and extract the results.

#### Task Explanations:
* The first task in the playbook creates a directory on the target servers where the UAC tool will be copied.
* The second task copies the UAC tool from the Ansible controller to the target servers. The version of the UAC tool to be used is determined by the variable "uac_version".
* The third task unarchives the UAC tool.
* The fourth task runs the UAC tool using the "command" module. The "creates" parameter is used to ensure that the UAC tool has completed running before moving on to the next task. The "timeout" parameter is set to 7200 seconds (2 hours) to allow for the UAC tool to run for as long as necessary.
* The fifth task creates a directory on the Ansible controller where the results of the UAC tool run will be copied.
* The sixth task recursively finds all files in the /uac/tmp directory on the target servers.
* The seventh task copies the resulting .log and .tar.gz files from the /uac/tmp directory on the target servers to the files/results directory on the Ansible controller. The files are copied with an IP address and timestamp added to the file name to ensure that they do not overwrite each other.
* The final task tells the user that the process is complete and that the results must be copied to the appropriate Box folder for the CSIRT team to inspect.
