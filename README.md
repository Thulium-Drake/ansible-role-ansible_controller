[![Build Status](https://drone.element-networks.nl/api/badges/Ansible/role-ansible_controller/status.svg)](https://drone.element-networks.nl/Ansible/role-ansible_controller)
# Controller setup
This role sets up a system with a dedicated Ansible user for use as a Ansible Control Node.

It will:
* Enable Ansible repository
* Install ansible, ansible-merge-vars, ARA client
* Download the Ansible Utils repository to your system (https://github.com/thulium-drake/ansible-utils)
* Create a new user
* Setup GPG agent for this user (Add your own GPG key, see below)

For EL7 you need to install GnuPG2.2 in order to fully use all of the tools this role provides. This package can
be found on https://copr.fedorainfracloud.org/coprs/icon/lfit/package/gnupg22-static/

And you need to enable EPEL on your system for some dependencies.

## Ansible Projects
The Controller uses the concept of 'Ansible Projects', these are folders containing everything Ansible
needs to do it's job. They should have the following structure:

```
.
|-- ansible.cfg                         # Ansible config
|-- cache                               # Fact cache
|-- collections                         # Collections required for the project
|-- files -> playbooks/files            # Convenience symlink to files
|-- group_vars -> inventory/group_vars  # Convenience symlink to inventory vars
|-- host_vars -> inventory/host_vars/   # Convenience symlink to inventory vars
|-- inventory                           # Directory with inventory files
|-- library                             # Any extra modules/plugins etc.
|-- playbooks                           # Playbooks
|-- requirements.yml                    # Ansible Galaxy requirements (roles and collections)
|-- roles                               # Roles required for the project
`-- scripts                             # Auxillary scripts for the project
```

The projects are stored in ```/opt/ansible/projects/<name-of-project>``` and are symlinked to the 'ansible' users'
homedir.

If a project has extra resources (which are not stored in Git), those can be found in ```/opt/ansible/resources```

### Ansible Vault
Most (if not all) Ansible projects use secrets, it is very important to save the secrets in a safe, encrypted
manner. Ansible provides ```ansible-vault``` for this purpose.

After creating a new project that has secrets (either encrypted vars files or encrypt_string vars in regular files),
do the following to set up the project:

* Set up retrieve_vault.sh

```
cd ~/projects/<name-of-project>
ln -s ../../vaults/retrieve_vault.sh .ansible_vault
echo 'my-ansible-vault-key' | gpg -e -r ansible@localhost > /opt/ansible/vaults/<name-of-project>
```

* Set the vault_password_file to .ansible_vault in ansible.cfg

### GPG Key
In order to allow for unattended runs, the 'ansible' user is set up to use GPG agent to hold a key once unlocked.
A cronjob that checks if any keys are locked is run once per hour, it will generate a cron mail when locked keys
are found.

In order to unlock the GPG key, run the ```gpgkey -u``` command and provide the passphrase.

To create a GPG key compatible for the Ansible Controller, follow the procedure below:

First you need a new (sub)key, note that if you want to add a EC key, your master key must be EC as well.

* Generate a new key (if you already have one, skip these steps)
 gpg --expert --full-gen-key
* Select 'ECC and ECC'
 9
* Select 'Curve 25519'
 1
* Set the expiration date (below sets the key to never expire)
 0
* Accept the notification that the key will never expire
 y
* Put in your name, email address and a comment (if you so desire)
 Ansible
 ansible@localhost
 Ansible Controller GPG key
* Accept the changes
 o
* Type in a strong passphrase

Now add a new subkey with the Authenticate flag

* Open the GPG utility
 gpg --expert --edit-key KEYID
* Add a new subkey
 addkey
* Select 'ECC (set your own capabilities)'
 11
* Disable 'Sign'
 s
* Enable 'Authenticate'
 a
* Exit the menu
 q
* Select 'Curve 25519'
 1
* Set the expiration date (below sets the key to never expire)
 0
* Accept the notification that the key will never expire
 y
* We really are sure...
 y
* Put in your passphrase

* Create a new backup of the GPG keys
 gpg --armor --export-secret-keys > gpg-key.txt


## Running Ansible jobs
After preparing the Controller, you can run Ansible jobs as follows:

```
Usage: /usr/local/bin/runansible [-p project] [-i inventory] [-s] playbook.yml

    -g           Update git checkout before running
    -p project   Name of the project to run the playbook from
    -i project   Name of the inventory to use, can be provided multiple times
    -r           Update roles before running playbook
    -s           Make Ansible's output sparse
    -h           This text
```

You can run this script with cron rules like the following:
```
0 0 * * * /usr/local/bin/runansible -sp <name-of-project> playbook.yml
```

The -s flag will reduce all output from playbook runs to stuff that has changed or is failed. Very useful for cron mails :-)
