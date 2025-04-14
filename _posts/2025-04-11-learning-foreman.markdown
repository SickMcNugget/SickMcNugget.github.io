---
layout: post
title:  "Learning Foreman"
date:   2025-04-11 09:52:48 +08:00
categories: tutorial linux sysadmin
published: true
---
# A Word on Foreman
[Foreman][foreman] is a tool that can provision enterprise-scale linux deployments. Foreman with the Katello plugin also allows for repositories to be synchronised and distributed to all host machines managed by foreman. 

The tool is so useful that Red Hat repackage it with their own fancy colour scheme and call it [Red Hat Satellite][rhs]. For my own sake, I'd like to go through, from start to finish, how I would provision a small intranet using foreman. 

The tutorials online frankly suck, and the user interface leaves a lot to be desired. My coworker says that the UI was definitely designed by a software engineer, and I'd have to agree. The upside is that I haven't had any bugs so far with the software, considering the scope of the project.

# Let's Get Into It
Foreman needs to do the following for my organisation:
- Provide IP addresses for multiple subnets
- Supply local DNS records
- Setup a firewall
- Provide time to all servers (NTP)
- Configure Switches
- Deploy Bare-Metal Machines (with iPXE)
- Deploy Virtual Machines (on top of Libvirt/oVirt)
- Provision SSH Keys
- Automate kernel/package updates (errata)
- Automate common CLI tasks across multiple hosts

Currenly, to do all this, my organisation uses a mixture of ansible, with an inventory provided by a home grown web server with some simple CRUD actions to add machines/roles, a bunch of web apps running using docker compose (and managed by a custom bash script), and nginx as a reverse proxy for the web apps. While everything works, pretty well, too I might add, I'd prefer a less obtuse solution, which is where foreman comes in.

## Installation
Installing foreman requires 4CPUs and 20GB of RAM, minimum. This feels excessive, but foreman is designed with enterprise clients in mind.

Since we're using katello, [these instructions][foreman-katello-install] are the ones to follow for setup. Before you run the installer, it's worth quickly explaining how to configure the installation to your liking.

### Configuration
There are some parameters that foreman doesn't allow configuring from it's web interface, so they need to be configured during the install process. One example is the storage location of `pulp`, a plugin that caches software so that it can be redistributed from the cache server. By default, the storage location is set to `/var/lib/pulp`. On the machine we use, our largest partition is `/home`, so we needed to move the pulp storage location to `/home/foreman/pulp`.

To do so requires 2 things. First, we need to understand two things:
- Puppet, the tool used to install and deploy foreman (and its plugins)
- The `/etc/foreman-installer/custom-hiera.yaml` file, which contains global configuration overrides for the foreman installer.

Since we are modifying the `pulpcore` module, we need to find it's reference documentation, which is found at [Puppetlabs Forge][pulpcore-reference]. Within this reference documentation, we see a parameter named `user_home`, which is set to `/var/lib/pulp`. This looks like the directory we're looking for!

To change it, we edit `/etc/foreman-installer/custom-hiera.yaml`.
```yaml
---
# This YAML file lets you set your own custom configuration in Hiera for the
# installer puppet modules that might not be exposed to users directly through
# installer arguments.
#
# For example, to set 'TraceEnable Off' in Apache, a common requirement for
# security auditors, add this to this file:
#
#   apache::trace_enable: 'Off'
#
# Consult the full module documentation on http://forge.puppetlabs.com,
# or the actual puppet classes themselves, to discover options to configure.
#
# Do note, setting some values may have unintended consequences that affect the
# performance or functionality of the application. Consider the impact of your
# changes before applying them, and test them in a non-production environment
# first.
#
# Here are some examples of how you tune the PostgreSQL options if needed:
#
# postgresql::server::config_entries:
#   max_connections: 600
#   shared_buffers: 1024MB

### Add the below line ###
pulpcore::user_home: "/home/foreman/pulp"
```
The module name you are looking for (in this case `pulpcore::`) can be found in the reference documentation.

If you have enough CPU/RAM, the only other problem you might run into is an invalid hosts file. This is a pretty quick fix. 

`/etc/hosts` when it only contains a host alias
```
10.0.0.2 foremanserver
```
```bash
# Check the hostname
hostname -f
# foremanserver
```
Let's fix it by adding an FQDN. 

`/etc/hosts` when it contains a domain and an alias
```
10.0.0.2 foremanserver.foremandomain foremanserver
```
```bash
# Check the hostname
hostname -f
# foremanserver.foremandomain
```

### Back to installing
After configuring, the install should work properly.

If, for some reason you lose access to stdout after the install completes, a log of the install is kept at `/var/log/foreman-installer/katello.log`. You can retrieve your admin credentials with grep
```bash
grep admin -r /var/log/foreman-installer
/var/log/foreman-installer/katello.20250410-171932.log:foreman::params::oauth_effective_user: admin
/var/log/foreman-installer/katello.20250410-171932.log:foreman::params::initial_admin_password: WM2LiTaAs9Q7tdV6

# No, the password isn't the same one we used
```

By default, foreman serves on port 80/443 (HTTP/HTTPS) and listens to all connections. Check the firewall on your server if you can't connect to the web interface.

<figure>
<image src="/assets/images/foreman/foreman_login.png" style="display: block; margin-left: auto; margin-right: auto; width: 60%"></image>
  <figcaption style="text-align: center;">The Foreman Login Screen</figcaption>
</figure>

## Adding Repositories
Let's start with something simple. We want repositories hosted on foreman that can be distributed to all the machines that we manage.

Head to **Content->Products**
<figure>
<image src="/assets/images/foreman/products_before.png" style="display: block; margin-left: auto; margin-right: auto; width: 60%"></image>
  <figcaption style="text-align: center;">The Products Page (I've already added some Rocky 9 repositories).</figcaption>
</figure>

Click on **Repo Discovery** in the top right.
Choose a repository to discover. For example, paste the rocky 8 mirror https://rockylinux.mirror.digitalpacific.com.au/8/ into the **URL to Discover** box, then hit **Discover**. After a minute or so, the repositories should be found.

Add the repositories that you want foreman to manage. I added AppStream, BaseOS, Devel and extras.
<figure>
<image src="/assets/images/foreman/repo_discovery.png" style="display: block; margin-left: auto; margin-right: auto; width: 100%"></image>
  <figcaption style="text-align: center;">Rocky 8 repository discovery (the BaseOS, Devel and extras repostories are off screen)</figcaption>
</figure>

Finally, press the **Create Selected** button.

In the new menu:
- change the **Product** selector to *New Product*
- Set the **Name** field to something relevant (I chose Rocky 8)
- Then press **Run Repository Creation**

<figure>
<image src="/assets/images/foreman/repo_creation.png" style="display: block; margin-left: auto; margin-right: auto; width: 100%"></image>
  <figcaption style="text-align: center;">Rocky 8 repository creation</figcaption>
</figure>

This will create the repositories from foreman's point of view, but they are not synced yet. Now is a good time to set your pulp3 storage directory, since that will be used when syncing the repositories.

**Updating pulp's storage location**
If you didn't reconfigure pulp's storage location in the **Configuration** section above, you can still change it now, at least until you run foreman-install again.

The `/etc/pulp/settings.py` file contains all the settings relating to where pulp stores data. You can configure it using the block below
```bash
# The below command could potentially break your settings file, so back it up!
pulp_dir="/home/foreman/pulp"
sed -i \
  -e 's|^\(MEDIA_ROOT = \)"\(.*\)"$|\1"'"$pulp_dir/media"'"   #\2|g' \
  -e 's|^\(FILE_UPLOAD_TEMP_DIR = \)"\(.*\)"$|\1"'"$pulp_dir/tmp"'"   #\2|g' \
  -e 's|^\(WORKING_DIRECTORY = \)"\(.*\)"$|\1"'"$pulp_dir/tmp"'"   #\2|g' \
  -e 's|^\(STATIC_ROOT = \)"\(.*\)"$|\1"'"$pulp_dir/assets"'"   #\2|g' \
  /etc/pulp/settings.py

# Make sure the directory exists with the correct permissions, too
mkdir -p "$pulp_dir/media" "$pulp_dir/tmp" "$pulp_dir/assets"
chown -R pulp:pulp "$pulp_dir"

# Reload pulpcore
foreman-maintain service restart --only pulpcore*
```

**Back to it**
Now navigate to the **Content->Products->Rocky 8** page. Tick all the repositories and click the **Sync Now** button. This downloads all the repositories you selected earlier. This caching allows you to distribute packages from the foreman server to many clients at once.

That's it for adding repositories. You can also add software directly from an ansible collection, a deb repository, a docker registry, a PyPI package, or directly from a file URL.

## Creating a VM managed by foreman
By default, foreman doesn't come shipped with any virtualisation providers. But adding them is as simple as installing a plugin from the foreman repository that we added earlier in the installation.

We recommend the use of libvirt, since we've generally had pretty good experiences with it, so we're going to use it for this tutorial.

```bash
dnf -y install foreman-libvirt
foreman-maintain service restart
```

### Giving foreman SSH access
As of foreman v3.14.0, the `/usr/share/foreman/.ssh` directory exists by default. 

This makes it easy to create some SSH keys for the `foreman` user, in the case where the VMs we are creating live on a remote host.

```bash
su foreman -s /bin/bash
ssh-keygen
ssh-copy-id  root@remotehost.remotedomain
ssh root@remotehost.remotedomain
exit
```

### Giving foreman local libvirt access
Foreman needs to be a member of the `libvirt` group to add VMs locally:
```bash
# As root
usermod -aG libvirt foreman
```

### Creating a Compute Resource
Navigate to **Infrastructure->Compute Resources** and click **Create Compute Resource**. Then, in order:
1. Fill in the name
2. Set the Provider to Libvirt
3. Fill in the QEMU connection URL (use qemu:///system for local)
4. Set the display type to VNC
5. Untick the *Console Passwords* box
6. Press the **Submit** button

> NOTE: I've changed this section since the foreman install we did earlier does not (by default) allow for PXE booting. DNS, DHCP and TFTP settings are disabled.
### Adding a Compute Profile
First, make sure that you have a valid .qcow2 disk so we can use it for provisioning.
Go to **Infrastructure->Compute Resources-><Your Compute Resource>->Images** and press the **Create Image** Button
1. Give the image a name
2. Select the OS
3. Set the Username and password for SSH access
4. Provide an image path. (This image should exist at that path on the REMOTE machine)
5. Press the **Submit** button

Now we create a compute profile by navigating to **Infrastructure->Compute Profiles** and clicking the **Create Compute Profile Button**.
1. Name the Compute Profile
2. Select the *Compute Resource* you wish to modify
3. Set the number of CPUs and amount of Memory
4. Select the image we just created
5. Change the network type to NAT
6. Set the storage pool (if available) type to QCOW2
<!-- ### Ensuring an Operating System is Available -->
<!-- Go to **Hosts->Provision Setup->Operating Systems**. You should see Rocky_Linux 9.5 in there. Click on that Operating System. In the **Installation Media** tab, Ensure that Rocky Linux is in the **Selected items** box. Learning all the settings in this section is left as an exercise to the reader. -->

### Creating the VM
Navigate to **Hosts->Create Host**. Then, in order:
1. Choose a name, organization and location
2. Set **Deploy On** to the Compute Resource we created earlier.
3. Set the **Compute Profile** to the Compute Profile we just created.
4. Set the content source, lifecycle environment and content view as required.
5. Navigate to the **Virtual Machine** tab. Don't worry if there is a warning about a missing storage pool. It will be created for us after we've done one VM. Alternatively, you could create on using `virsh`.
5. Set the number of CPUs and the amount of Memory you want.
6. Navigate to the **Operating System** tab.
7. Set the following options
```
Architecture: x86_64
Operating System: Rocky_Linux 9.5
Provisioning Method: Image Based
Build Mode: Ticked
Media Selection: All Media
Media: Rocky Linux
Partition Table: Kickstart default
PXE loader: PXELinux BIOS
Root Password: Your choice
```
8. Navigate to the **Interfaces** tab
9. Edit the default interface and set the following:
```
Domain: your domain
```
10. Press **Ok** and then **Submit**

> Currently doesn't work. I've had to move over to some other tasks but I'll fill this out further later on.

<!-- ### Allowing Connection without Auth or Encryption (Optional) -->
<!-- in `/etc/libvirt/libvirtd.conf` set the following option: -->
<!-- ``` -->
<!-- listen_tls = 0 -->
<!-- listen_tcp = 1 -->
<!-- auth_tcp = "none" -->
<!-- ``` -->



[foreman]: https://theforeman.org/
[rhs]: https://www.redhat.com/en/technologies/management/satellite
[foreman-katello-install]: https://docs.theforeman.org/3.14/Quickstart/index-katello.html
[pulpcore-reference]: https://forge.puppetlabs.com/modules/theforeman/pulpcore/reference
