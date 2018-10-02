Ansible Role: KSbuilder
=======================

The idea of this role is to build a system based on a variables passed to the
role in addition to a full binary ISO of an installation media and a kickstart
file.

I prefer to pass the variables "into" the role from the playbook versus by
includeing variable files.  This is because I hope to make the role usable by
other roles.  I don't know if this logic makes sense or not, but I am
essentially attempting to remove the variables from the role itself.  (Maybe I
should at least include defaults that simply don't work?  I think that if I put
them in a file in the repo, then others will assume that is where they should
go and that is not my intention.)

At a high level, this role does the following:

1. Create an nginx container to host the kickstart file
2. Put the kickstart file in the appropriate location
3. Do some fanagling to figure out the IP of the host system in the libvirt
   network specified (this is for use in the `ks=` part of `virt-install`)
4. Use `virt-install` to create the VM with all the specified configurations
   and by using the supplied ISO and kickstart file
5. Wait for the installation to finish (currently checking every 5 seconds,
   180 times for a total of 15 minutes before considered failed)
6. Stop the nginx container

The result is a freshly built, but shut off, VM.  The longterm vision is that
this role will be used in other roles...

Important Notes
---------------

* This is currently developed to be used on a system locally.  In order to use
  it with Tower or AWX will take some refactoring
* This role is far from idempotent at this point.  During testing I have found
  the following commands helpful to clean up:

      sudo docker rm -f nginxKS;
      sudo rm -vfr /tmp/KS;
      sudo virsh destroy <vm.name>;
      sudo virsh undefine <vm.name> --remove-all-storage

* My example playbook prompts the user for passwords to use in the VM...sort of
  breaking unattended automation, but that's how I do it right now.

Requirements
------------

I couldn't escape a few requirements:

* **docker**:  Currently this role is spinning up a nginx container in order to
  house the kickstart file.  I considered doing things like building a custom
  ISO with the KS file, but figured I'd just spin up a container because,
  selfishly I'm using them anyway.  But in addition to having docker installed
  and configured, you'll need these packages:
  * **python2-passlib**: *this is actually only necessary if you use the
    "encrypt" capability of  vars_prompt in your playbook...*
  * **python2-docker-py**
* You'll need to have your libvirt environment configured to your liking.  I'm
  working on creating my own preferences in a separate role...to be continued...
* A locally downloaded full binary ISO of the distribution you wish to install
  *[the need for a full binary ISO is dependant on how you use it in your own
  kickstart file]*
* A kickstart template (I'll have to think about this because I'll likely use
  this role in other roles so those will have to put the template in the correct
  location?)

Role Variables
--------------

All of these variables should be considered **required** however, it will
depend greatly on how you set up your kickstart file.  Just know that there is
currently no sanity checking:

* `ks_root_password`
  * this is needed for the kickstart template.  You can provide it securely in
    any number of ways such as by using a vault or by prompting for it in your
    playbook (see example below).  Keep in mind that this is the root password for your new VM
* `ks_ansible_password`
  * this is only needed if you creata an ansible user in your kickstart
    template.  I tend to do this.
* `vm` *I've created a bit of a nested list here so that the variables can be
  used like `vm.name`*
  * `name`
    * the hostname of the new VM
  * `disk` *the next layer of this nest*
    * `pool`
      * the libvirt storage pool the VM should be created in
    * `size`
      * the size of the disk assigned to the VM ***(in GB)***
  * `vcpus`
    * the number of VCPUs assigned to the VM
  * `memory`
    * the amount of memory assigned to the VM ***(in MB)***
  * `network` *another layer, but with the idea that I could add IP info, etc*
    * `name`
      * the libvirt network the VM will be on
* `iso`
  * `name`
    * the filename of the ISO to be attached to the VM for installation
  * `location`
    * the absolute path to the ISO to be used ***this is local to the system
      calling the playbook***
* `ks`
  * `name`
    * the filename of the kickstart template to use (there are some examples in
      the role repository) *perhaps use `delegate_to` in another role to put a
      role-specific ks template in **this** role's template directory?*
  * `use_container` ***NOT USED***
    * I should work in the ability to specify the use of the container...in due
      time

Example Playbook
----------------

Playbook with configuration options specified:

```yaml
- hosts: localhost
  connection: local
  vars_prompt:
    - name: "ks_root_password"
      prompt: "Enter password for new VM"
      private: yes
      encrypt: "sha512_crypt"
      confirm: yes
      salt_size: 7
    - name: "ks_ansible_password"
      prompt: "Enter password for ansible user in new VM"
      private: yes
      encrypt: "sha512_crypt"
      confirm: yes
      salt_size: 7
  roles:
    - role: KSbuilder
      vm:
        name: minimal
        disk:
          pool: default
          size: 15
        vcpus: 2
        memory: 512
        network:
          name: default # this is the libvirt network the vm will be on
      iso:
        name: rhel-workstation-7.5-x86_64-dvd.iso
        location: /depot/isos # this must be local
      ks:
        name: example-workstation.cfg.j2 # this must be in <role>/templates
        use_container: yes

```

To-do
-----

* I need to figure out the best way to "include" this role in other roles.  I'm
  imagining this role as a part of other roles, such as to create a Satellite
  server.  So look into [requirements.yml](https://docs.ansible.com/ansible/latest/reference_appendices/galaxy.html).
  * **UPDATE:** I have included a fairly generic `meta/main.yml` file which
    allows for something similar to:

        ansible-galaxy install -p ./roles -r requirements.yml

    with `requirements.yml` containing:

        ---
        # get the builder role from github
        - src: https://github.com/thisdwhitley/ansible-role-KSbuilder.git
          scm: git
          name: KSbuilder

References
----------

* [rickmanley-nc's deploy in satellite repo](https://github.com/rickmanley-nc/satellite)
* [bhirsch70's ansible-prvsn-libvirt-vm role](https://github.com/RedHatGov/Instant-Demo/tree/master/ansible-prvsn-libvirt-vm)
* [toshywoshy's work](https://github.com/toshywoshy/ansible-role-vminstaller)

License
-------

Red Hat, the Shadowman logo, Ansible, and Ansible Tower are trademarks or
registered trademarks of Red Hat, Inc. or its subsidiaries in the United
States and other countries.

All other parts of this project are made available under the terms of the [MIT
License](LICENSE).