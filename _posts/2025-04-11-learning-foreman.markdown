---
layout: post
title:  "Learning Foreman"
date:   2025-04-11 09:52:48 +08:00
categories: tutorial linux sysadmin
published: false
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

## Prerequisites
Installing foreman requires:
- 4CPUs 
- 20GB of RAM,
- 4GB RAM of swap space.
- en_US.utf-8 locale
- A unique hostname (can contain \[a-z0-9.-\])

This feels excessive, but foreman is designed with enterprise clients in mind. The RAM and CPU count is based on having up to 5000 machines managed by foreman.

### Storage

| Directory | Installation Size | Runtime Size |
| --------------- | --------------- | --------------- |
| /var/log | 10MB | 10GB |
| /var/lib/pgsql | 1GB | 20GB |
| /usr | 10GB | NA |
| /opt/puppetlabs | 500MB | NA |
| /var/lib/pulp | 1MB | 300GB |

**Do not use symlinks for /var/lib/pulp/**

They recommend using **logrotate** for reducing log storage requirements.
Consider mount `/var` on LVM storage (most foreman server data is stored on /var).
Use high-bw, low-latency storage for `/var/lib/pulp` and `/var/lib/pgsql`.
Use a low I/O latency filesystem (Not GFS2).

## Pre-Installation
### Opening Ports
```bash
# Ports for clients on Foreman server
firewall-cmd \
--add-port="8000/tcp" \
--add-port="9090/tcp"
```

```bash
# services on foreman-server
firewall-cmd \
--add-service=dns \
--add-service=dhcp \
--add-service=tftp \
--add-service=http \
--add-service=https \
--add-service=puppetmaster
```

```bash
# Make the changes persistent
firewall-cmd --runtime-to-permanent
```

You can check the above commands worked with:
```bash
firewall-cmd --list-all
```

### Verifying DNS Resolution
First, check whether hostname and localhost resolve properly
```bash
ping -c1 localhost
ping -c1 `hostname -f`
```
If your hostname fails, update it to the one you want:
```bash
# Check current hostname
hostnamectl
# Set new hostname
hostnamectl hostname master-server.management
```

Now, try pinging again.
If that still fails, update your `/etc/hosts` file so that the server recognises itself.

```
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.2 master-server.management master-server
```
Pinging should now work, assuming you set the IP correctly in `/etc/hosts`

### Create our Bridge Interface
We'll use a bridge interface for our DHCP and DNS servers. It also allows us to connect VMs to this bridge via a libvirt network later.

```bash
nmcli con add type bridge ifname br200 con-name br200
nmcli con modify br200 bridge.stp no
nmcli con add type bridge-slave ifname enp130s0f0np0 master br200
nmcli con up br200
nmcli con up bridge-slave-enp130s0f0np0

nmcli con modify br200 ipv4.method manual ipv4.addr "192.168.200.3/24" ipv4.dns "192.168.200.2" ipv4.gateway "192.168.200.1"
```

## Installation
Since we're using katello, the [quickstart instructions][foreman-katello-3.16-quickstart] and the [in-depth instructions][foreman-katello-3.16-indepth] are the ones to follow for this guide.

Try to complete all the steps from the quickstart now (following the puppet steps), except for running:
```bash
foreman-installer --scenario katello
```
since we have some configuration options first.

### Configuration
#### Configuring pulp (Optional)
There are some parameters that foreman doesn't allow configuring from its web interface, so they need to be configured during the install process. One example is the storage location of `pulp`, a plugin that caches software so that it can be redistributed from the cache server. By default, the storage location is set to `/var/lib/pulp`. On the machine we use, our largest partition is `/home`, so we needed to move the pulp storage location to `/home/foreman/pulp`.

To do so requires 2 things. First, we need to understand two things:
- Puppet, the tool used to install and deploy foreman (and its plugins)
- The `/etc/foreman-installer/custom-hiera.yaml` file, which contains global configuration overrides for the foreman installer.

Since we are modifying the `pulpcore` module, we need to find its reference documentation, which is found at [Puppetlabs Forge][pulpcore-reference]. Within this reference documentation, we see a parameter named `user_home`, which is set to `/var/lib/pulp`. This looks like the directory we're looking for!

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

The modules can also be found in the `foreman-installer` Puppetfile, located [here](https://github.com/theforeman/foreman-installer/blob/develop/Puppetfile).

### Back to installing

#### Fixing broken dnf installs (Optional)
> We couldn't get this to work! We tried to install foreman on Rocky 10, but we had a problem with dnf being unable to find a suitable python 3.9. I'm sure there's a way to tell dnf that we do actually have a valid python version, and where it is, but that's left to the reader.

If you're anything like me, you've tried to install foreman on Rocky 10 even though it explicitly says that it only supports el9. Regardless, I'm going to try fix this.
The first problem is that it needs python 3.9. Let's get that.

Download pyenv:
```bash
curl -fsSL https://pyenv.run | bash

cat <<-'EOF' >>~/.bashrc
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - bash)"
EOF

cat <<-'EOF' >>~/.bash_profile
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - bash)"
EOF
```

Restart your shell now
```bash
# Install build tools for getting the correct python version
dnf -y group install "Development Tools"
pyenv install 3.9
pyenv global 3.9
```

#### Running the installer
After configuring, you can now check the installer options:
```bash
# Understand what your options are
foreman-installer --scenario katello --help
# REALLY understand what your options are
foreman-installer --scenario katello --full-help
```

You can try an unattended installation:
```bash
# Don't forget that you can add all your own configuration options here.
foreman-installer --scenario katello 
```
Or an interactive installation:
```bash
foreman-installer -i
```

#### Interactive install choices
I've chosen not to enable apache_mod_status as it was off by default.
I've chosen to enable certs as it was on by default.
I've chosen to enable foreman as it was on by default.
I've chosen to enable foreman_cli as it was on by default.
I've chosen to enable foreman_cli_ssh (SSH commands via CLI to remote hosts).

I've chosen not to enable foreman_compute_ec2 as we don't use EC2.
I've chosen to enable foreman_compute_libvirt as we use libvirt.
I've chosen not to enable foreman_compute_openstack as we don't use OpenStack.
I've chosen not to enable foreman_compute_vmware as we don't use VMWare.


I've chosen to enable foreman_plugin_ansible, foreman_cli_ansible and foreman_proxy_plugin_ansible (for running playbooks on hosts)
I've chosen not to enable foreman_plugin_azure and foreman_cli_azure (as we don't use azure).
I've chosen to enable foreman_plugin_bootdisk and foreman_cli_bootdisk (Whilst foreman supports DHCP+TFTP-based PXE booting by default, this also allows iPXE-style booting, instructions [here](https://github.com/theforeman/foreman_bootdisk))
I've chosen to enable foreman_plugin_discovery, foreman_cli_discovery and foreman_proxy_plugin_discovery and foreman_proxy_plugin_discovery_install_images=true (discovery of bare-metal hosts which can be registered with foreman).
I've chosen to enable foreman_plugin_dhcp_browser (allows you to edit DHCP from the browser).
I've chosen not to enable foreman_plugin_dlm (Lets clients acquire a lock so only one server updates the operating system at a time).
I've chosen to enable foreman_plugin_expire_hosts (allows you to set an expiry date for a host, so that it disables and deletes when needed).
I've chosen not to enable foreman_plugin_git_templates (add support for using templates from git repositories).
I've chosen not to enable foreman_plugin_google, foreman_cli_google (google compute engine support?)
I've chosen not to enable foreman_plugin_hdm and foreman_proxy_plugin_hdm (some hiera data thing).
I've chosen not to enable foreman_plugin_kernel_care (live patch linux kernels without system reboot).
I've chosen not to enable foreman_plugin_kubevirt and foreman_cli_kubevirt (running a VM inside of a kubernetes pod).
I've chosen not to install foreman_plugin_leapp (in-place system upgrades for RHEL), since we use more than just RHEL.
I've chosen to install foreman_plugin_monitoring and foreman_proxy_plugin_monitoring (server status monitoring).
I've chosen not to install foreman_plugin_netbox (integration with NetBox Configuration Management DB (CMDB))
I've chosen not to install foreman_plugin_openscap and foreman_proxy_plugin_openscap (vulnerability assessment and compliance audit of foreman infrastructure).
I've chosen not to install foreman_plugin_proxmox (Proxmox-based VM provisioning), since my team has no use for proxmox.
I've chosen not to install foreman_plugin_puppet, foreman_plugin_puppetdb, foreman_cli_puppet and puppet (infrastructure tool that can handle the same functionality as ansible).
I've chosen to enable foreman_plugin_remote_execution, foreman_cli_remote_execution and foreman_proxy_remote_execution_script and foreman_plugin_remote_execution_cockpit (run commands remotely on hosts in the infrastructure).
I've chosen to enable foreman_plugin_rescue (PXE boot a foreman host into a rescue system).
I've chosen not to enable foreman_plugin_resource_quota and foreman_cli_resource_quota (limit access to shared resources by giving a fixed number of resources to users/groups).
I've chosen not to enable foreman_plugin_rh_cloud, due to a lack of documentation.
I've chosen not to enable foreman_salt and foreman_proxy_plugin_salt (a similar tool to ansible), since we've already enabled the use of ansible. Although salt does look like a cool alternative. (Note that 3.16 doesn't have foreman_cli_salt, but newer versions do!)
I've chosen not to enable foreman_plugin_scc_manager (sync SUSE customer center products and repos into Katello).
I've chosen not to enable foreman_plugin_snapshot_management (manage vSphere and proxmox snapshots from foreman UI) since we don't use vSphere or proxmox.
I've chosen not to enable foreman_plugin_statistics (lets foreman show a nice statistics dashboard and create trend charts), since we just don't need the stats.
I've chosen to enable foreman_plugin_tasks and foreman_cli_tasks (run, monitor, debug and audit jobs, processes, actions. A core component).
I've chosen to enable foreman_plugin_templates and foreman_cli_templates (it seems to provide more templates for generating resources, but I'm unsure if it is needed with v3.16 of foreman).
I've chosen not to enable foreman_plugin_vault (integration with HashiCorp Vault secret manager).
I've chosen not to enable foreman_plugin_virt_who_configure and foreman_cli_plugin_virt_who_configure, as that seems to have something to do with "reporting VM statistics to a subscription manager" like Red Hat Subscription Management (RHS) or Satellite 6
I've chosen to enable foreman_plugin_webhooks, foreman_cli_webhooks and foreman_proxy_plugin_shellhooks (Allows webhooks to be configured for Foreman).
I've chosen not to enable foreman_plugin_wreckingball (vSphere status checks) since we don't use vSphere.

I've chosen to enable foreman_proxy as it was on by default.
I've chosen to enable foreman_proxy_content as it was on by default.

I've chosen not to enable foreman_proxy_plugin_dhcp_infoblox (infoblox DHCP provider support).
I've chosen not to enable foreman_proxy_plugin_dhcp_remote_isc (support interfacing with ISC-dhcpd servers over NFS).
I've chosen not to enable foreman_proxy_plugin_dns_infoblox (infoblox DNS provider support).
I've chosen not to enable foreman_proxy_plugin_dns_powerdns (PowerDNS DNS provider support).
I've chosen not to enable foreman_proxy_plugin_dns_route53 (Amazon Route 53 DNS provider support).

I've chosen not to enable iop since I don't know what it does.
I've chosen to enable katello and foreman_cli_katello (does everything).
I've chosen not to enable puppet since I don't know what it does.

To put this into an automated install:
```bash
foreman-installer --scenario=katello \
  --no-enable-apache-mod-status \
  --enable-certs \
  --enable-foreman \
  --enable-foreman-cli \
  --enable-foreman-cli-ssh \
  --no-enable-foreman-compute-ec2 \
  --enable-foreman-compute-libvirt \
  --no-enable-foreman-compute-openstack \
  --no-enable-foreman-compute-vmware \
  --enable-foreman-plugin-ansible --enable-foreman-cli-ansible --enable-foreman-proxy-plugin-ansible \
  --no-enable-foreman-plugin-azure --no-enable-foreman-cli-azure \
  --enable-foreman-plugin-bootdisk --enable-foreman-plugin-bootdisk \
  --enable-foreman-plugin-discovery --enable-foreman-cli-discovery --enable-foreman-proxy-plugin-discovery --foreman-proxy-plugin-discovery-install-images=true \
  --enable-foreman-plugin-dhcp-browser \
  --no-enable-foreman-plugin-dlm \
  --enable-foreman-plugin-expire-hosts \
  --no-enable-foreman-plugin-git-templates \
  --no-enable-foreman-plugin-google \
  --no-enable-foreman-plugin-hdm \
  --no-enable-foreman-plugin-kernel-care \
  --no-enable-foreman-plugin-kubevirt --no-enable-foreman-cli-kubevirt \
  --no-enable-foreman-plugin-leapp \
  --enable-foreman-plugin-monitoring --enable-foreman-proxy-plugin-monitoring \
  --no-enable-foreman-plugin-netbox \
  --no-enable-foreman-plugin-openscap --no-enable-foreman-proxy-plugin-openscap \
  --no-enable-foreman-plugin-proxmox \
  --no-enable-foreman-plugin-puppet \
  --enable-foreman-plugin-remote-execution --enable-foreman-cli-remote-execution --enable-foreman-proxy-plugin-remote-execution-script --enable-foreman-plugin-remote-execution-cockpit \
  --enable-foreman-plugin-rescue \
  --no-enable-foreman-plugin-resource-quota --no-enable-foreman-cli-resource-quota \
  --no-enable-foreman-plugin-rh-cloud \
  --no-enable-foreman-plugin-salt --no-enable-foreman-proxy-plugin-salt \
  --no-enable-foreman-plugin-scc-manager \
  --no-enable-foreman-plugin-snapshot-management \
  --no-enable-foreman-plugin-statistics \
  --enable-foreman-plugin-tasks --enable-foreman-cli-tasks \
  --enable-foreman-plugin-templates --enable-foreman-cli-templates \
  --no-enable-foreman-plugin-vault \
  --no-enable-foreman-plugin-virt-who-configure --no-enable-foreman-cli-virt-who-configure \
  --enable-foreman-plugin-webhooks --enable-foreman-cli-webhooks --enable-foreman-proxy-plugin-shellhooks \
  --no-enable-foreman-plugin-wreckingball \
  --enable-foreman-proxy \
  --enable-foreman-proxy-content \
  --no-enable-foreman-proxy-plugin-dhcp-infoblox \
  --no-enable-foreman-proxy-plugin-dhcp-remote-isc \
  --no-enable-foreman-proxy-plugin-dns-infoblox \
  --no-enable-foreman-proxy-plugin-dns-powerdns \
  --no-enable-foreman-proxy-plugin-dns-route53 \
  --no-enable-iop \
  --enable-katello --enable-foreman-cli-katello \
  --no-enable-puppet
```

**Addendum**
By default, this does not enable DNS/DHCP/TFTP management via the foreman smart proxy (remember one is integrated with foreman server), so we need to also enable those options.

We can also allow HTTPBoot, too, in case we want that. Unless we also enable the http_port for smart_proxy, the UEFI boot will have to trust the smart-proxies' certificate (I think).

The configuration options can be found on the [Integrating provisioning infrastructure services](https://docs.theforeman.org/3.16/Integrating_Provisioning_Infrastructure_Services/index-katello.html) page on the foreman docs.

```bash
foreman-installer -l DEBUG \
--foreman-proxy-dns true \
--foreman-proxy-dns-forwarders "1.1.1.1" \
--foreman-proxy-dns-provider nsupdate \
--foreman-proxy-dns-managed true \
--foreman-proxy-dns-interface br200 \
--reset-foreman-proxy-dns-server \
--foreman-proxy-dhcp true \
--foreman-proxy-dhcp-provider isc \
--foreman-proxy-dhcp-managed true \
--foreman-proxy-dhcp-range "192.168.200.100 192.168.200.150" \
--foreman-proxy-dhcp-gateway 192.168.200.1 \
--foreman-proxy-dhcp-nameservers "192.168.200.2" \
--foreman-proxy-dhcp-interface br200 \
--reset-foreman-proxy-dhcp \
--foreman-proxy-tftp true \
--foreman-proxy-tftp-managed true \
--foreman-proxy-tftp-servername 192.168.200.3 \
--reset-foreman-proxy-tftp-servername \
--foreman-proxy-httpboot true \
--foreman-proxy-http true
```

For HTTPBooting, you need to enter the subnet that you are booting the machine into, and then choose the corresponding HTTPBoot proxy for that subnet.

**Infrastructure->Subnets->temporary-200-network-for-testing->Proxies->HTTPBoot Proxy = master-server.management**



If, for some reason you lose access to stdout after the install completes, a log of the install is kept at `/var/log/foreman-installer/katello.log`. You can retrieve your admin credentials with grep
```bash
grep admin -r /var/log/foreman-installer
# /var/log/foreman-installer/katello.20250410-171932.log:foreman::params::oauth_effective_user: admin
# /var/log/foreman-installer/katello.20250410-171932.log:foreman::params::initial_admin_password: WM2LiTaAs9Q7tdV6

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

This will create the repositories from foreman's point of view, but they are not synced yet. If you've configured your pulp3 storage directory incorrectly, you can temporarily change it until you run `foreman-install` again. Refer to appendix A.


**Back to it**
Now navigate to the **Content->Products->Rocky 8** page. Tick all the repositories and click the **Sync Now** button. This downloads all the repositories you selected earlier. This caching allows you to distribute packages from the foreman server to many clients at once.

That's it for adding repositories. You can also add software directly from an ansible collection, a deb repository, a docker registry, a PyPI package, or directly from a file URL.

## Creating a VM managed by foreman
Since we installed the foreman-libvirt plugin with foreman-installer, this functionality should already be available to us. Let's start by creating a local VM that runs OPNSense, to act as our firewall.

Start by ensuring that libvirt is installed and running on the foreman host:
```bash
dnf install -y libvirt qemu-kvm
systemctl enable --now libvirtd
```

### Giving foreman libvirt permissions
To be able to create a VM on the same host that foreman is running on, the foreman user needs permission to perform privileged libvirt operations. Lets provide that permission:
```bash
usermod -aG libvirt foreman
```

### Creating a storage pool
We want a storage pool to be available from the get-go when creating our VM. Let's make one now:
```bash
virsh pool-define-as default dir --target /mnt/foreman/libvirt-pool
virsh pool-start --build default
virsh pool-autostart default
virsh pool-list
#  Name      State    Autostart
# -------------------------------
#  default   active   yes
```

### Creating a Compute Resource
We need to create a compute resource that represents the hypervisor we want to use for creating a VM.

Navigate to **Infrastructure->Compute Resources** and click **Create Compute Resource**. Then, in order:
1. Fill in the name
2. Set the Provider to Libvirt
3. Fill in the QEMU connection URL (use qemu:///system for local)
4. Set the display type to VNC
5. Untick the *Console Passwords* box
6. Press the **Submit** button

### Creating an OS entry for OPNSense
Navigate to **Hosts->Provisioning Setup->Operating Systems**. Then, in order:
1. Press **Create Operating System**
2. Set the following options:
```
Name: OPNSense
Major Version: 25
Minor Version: 7
Description: OPNSense 25.7
Family: FreeBSD
Root Password Hash: SHA512
Architectures: x86_64
```
3. Press **Submit**

### Create the VM
Navigate to **Hosts->Create Host**. Then, in order:
1. Choose a name, organization and location
2. Set **Deploy On** to the Compute Resource we created earlier.
3. Set the **Compute Profile** to the Compute Profile we just created.
4. Set the content source, lifecycle environment and content view as required.
5. Set the monitoring proxy to the only one available.
6. Navigate to the **Virtual Machine** tab and set the amount of CPU and memory you want. Also set the size and type of storage you want for the VM (I use QCOW2).
7. Set the number of CPUs and the amount of Memory you want.
8. Navigate to the **Operating System** tab.
9. Set the following options
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

## Notes
From []
> Foreman server has an integrated Smart Proxy and any host that is directly connected to Foreman server is a Client of Foreman in the context of this section. This includes the base operating system on which Smart Proxy server is running.

# Appendix A
## Updating pulp's storage location
Remember, these changes only persist until you run `foreman-install`, at which point, you should really be modifying `/etc/foreman-install/custom-hiera.yaml`.

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

[foreman]: https://theforeman.org/
[rhs]: https://www.redhat.com/en/technologies/management/satellite
[foreman-katello-3.16-quickstart]: https://docs.theforeman.org/3.16/Quickstart/index-katello.html
[foreman-katello-3.16-indepth]: https://docs.theforeman.org/3.16/Installing_Server/index-katello.html#configuring-repositories_foreman
[pulpcore-reference]: https://forge.puppetlabs.com/modules/theforeman/pulpcore/reference
