# Ansible Role: KSbuilder

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
7. Inject ssh key for root into the shut off system (installing packages if
   needed)
8. Power on the VM
9. Add it to an *allVMs* group for use in future plays

The result is a freshly built VM populated with the ssh key of the user running
the role (for root login).  The longterm vision is that this role will be used 
in other roles...

## Important Notes

* This is currently developed to be used on a system locally.  In order to use
  it with Tower or AWX will take some refactoring
* This role is far from idempotent at this point.  During testing I have found
  the following commands helpful to clean up:

      sudo docker rm -f nginxKS;
      sudo rm -vfr /tmp/KS;
      sudo virsh destroy <vm.name>;
      sudo virsh undefine <vm.name> --remove-all-storage

* The use of multiple disks will require that your kickstart file handle this
  appropriately.  For example, if you specify multiple disks, and your kickstart
  uses `autopart`, your root LVM will be split across the disks and the mounting
  of the disk for SSH key injection will fail.  See
  `templates/example-server.cfg.j2` for an example of how I'm doing this.

## Requirements

I couldn't escape a few requirements:

* **docker**:  Currently this role is spinning up a nginx container in order to
  house the kickstart file.  I considered doing things like building a custom
  ISO with the KS file, but figured I'd just spin up a container because,
  selfishly I'm using them anyway.
* You'll need to have your libvirt environment configured to your liking.  I'm
  working on creating my own preferences in a separate role...to be continued...
* A locally downloaded full binary ISO of the distribution you wish to install
  *[the need for a full binary ISO is dependant on how you use it in your own
  kickstart file]*
* A kickstart template (I'll have to think about this because I'll likely use
  this role in other roles so those will have to put the template in the correct
  location?)

## Role Variables

All of these variables should be considered **required** however, it will
depend greatly on how you set up your kickstart file.  Just know that there is
currently no sanity checking:

* `vm` *I've created a bit of a nested list here so that the variables can be
  used like `vm.name`*
  * `name`
    * the hostname of the new VM
  * `disks` *the next layer of this nest*
    * `boot_order`
      * this is important when multiple disks are added (1-x)
    * `pool`
      * the libvirt storage pool the VM should be created in
    * `size`
      * the size of the disk assigned to the VM ***(in GB)***
  * `vcpus`
    * the number of VCPUs assigned to the VM
  * `memory`
    * the amount of memory assigned to the VM ***(in MB)***
  * `ip`
    * the IP address that will be set on the VM
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

## Example Playbook

Playbook with configuration options specified:

```yaml
---
- hosts: localhost
  connection: local
  roles:
    - role: KSbuilder
      vm:
        name: minimal7.labtop.local # use the FQDN
        disks:
          - boot_order: 1
            pool: default
            size: 16
          - boot_order: 2
            pool: default
            size: 100
        vcpus: 4
        memory: 12228
        ip: 192.168.1.2 # this must be within the network specified below
        network:
          name: default # this is the libvirt network the vm will be on
      iso:
        name: rhel-server-7.6-x86_64-dvd.iso
        location: /depot/isos # this must be local
      ks:
        name: example-server.cfg.j2
        use_container: yes
      tags: KSbuilder

```

## Inclusion

I envision this role being included in a larger project through the use of a
`requirements.yml` file.  So here is an example of what you would need in your
file:

```yaml
# get the KSbuilder role from github
- src: https://github.com/thisdwhitley/ansible-role-KSbuilder.git
  scm: git
  name: KSbuilder
```

Have the above in a `requirements.yml` file for your project would then allow
you to "install" it (prior to use in some sort of setup script?) with:

```bash
ansible-galaxy install -p ./roles -r requirements.yml
```

## Testing

The purpose of this project does not lend itself to easy testing.  It is kind
of crappy, sorry about that.

## References

* [rickmanley-nc's deploy in satellite repo](https://github.com/rickmanley-nc/satellite)
* [bhirsch70's ansible-prvsn-libvirt-vm role](https://github.com/RedHatGov/Instant-Demo/tree/master/ansible-prvsn-libvirt-vm)
* [toshywoshy's work](https://github.com/toshywoshy/ansible-role-vminstaller)

## License

Red Hat, the Shadowman logo, Ansible, and Ansible Tower are trademarks or
registered trademarks of Red Hat, Inc. or its subsidiaries in the United
States and other countries.

All other parts of this project are made available under the terms of the [MIT
License](LICENSE).
