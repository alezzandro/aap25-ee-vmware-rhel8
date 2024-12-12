# Custom Ansible Execution Environment in RHEL8 with VMware and OCP4 Collections

You can find below the commands needed for creating a custom Ansible Execution Environment in RHEL8 with several collections, including `OpenShift` / `OpenShift Virtualization` collections and the `community.vmware` collection.

This Ansible Execution Environment could be really useful for troubleshooting and automating stuff during [OpenShift Virtualization](https://docs.openshift.com/container-platform/4.16/virt/about_virt/about-virt.html) Proof of Concepts with [Migration Toolkit for Virtualization - MTV](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.7/html-single/installing_and_using_the_migration_toolkit_for_virtualization/index).

The main use cases for which I created this Execution Environment is the following but consider that you may also leverage it for any other automation demos on OCP Virt.

## VMware change disk mode for a vm from independent to dependent.

In VMware there are three [different kind of disk modes](https://www.bdrsuite.com/blog/what-are-the-different-disk-modes-in-vmware/) you can select when attaching a disk to a Virtual Machine (VM).

If you are planning to migrate with MTV a virtual machine with independent disks on OCP Virt, then you may hit this bug: 
```
 GetFileName: Cannot create disk spec for disk <name>.
 Error occurred when obtaining the file name for <name>.
```
If the Disk Mode is `Independent-Persistent` or `Independent-Nonpersistent`, then VDDK ≥ 7 has a bug where it cannot open these disks. You can try to leverage VDDK ≤ 6.7 (<b><u>but it didn't work for me</b></u>), or to change the Disk Mode to a regular dependent disk.
More info here in the [nbdkit-vddk-plugin page](https://libguestfs.org/nbdkit-vddk-plugin.1.html#The-guest-is-using-an-Independent-mode-disk).


For this reason changing the disk mode is mandatory to migrate a Virtual Machine with the Migration Toolkit for Virtualization. Of course for a massive migration changing manually every disk mode in VMware vSphere is not so handy, that's why we could leverage Ansible for this job.

In a Proof of Concept you can leverage the `ansible-playbook` command line tool through the Execution Environment container, but of course in a real production environment you can level up to Ansible Automation Platform (AAP) very easily also leveraging the resources of this repository.

## Building the Execution Environment

The first prerequisite is to have a RHEL8 correctly attached to Red Hat subscription and repositories. We will start from an minimal Execution Environment taken from the AAP container images adding also several other binaries needed for the various collection.

<b><u>Please note</u></b>: the collection `community.vmware` doesn't not strictly require RHEL8 base image, so if you want you could change the collection requirements and the execution environment accordingly for leveraging upstream versions.

After cloning this repository on your RHEL8 system, you have to install at least the following packages:
```
# subscription-manager repos --enable ansible-automation-platform-2.5-for-rhel-8-x86_64-rpms
# dnf install podman ansible-builder ansible-navigator -y

```

Then we are ready to fill the latest configuration in the `execution-environment.yml` config file, go to https://console.redhat.com/ansible/automation-hub/token load/copy your token and paste it in the file:
```
    - ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN=___YOUR___TOKEN___
```

Before running the build you can also take a look to the `bindep.txt` file in case you may want/need to add additional packages to the final Execution Environment image. I've already included some text editor and tools:
```
systemd-devel
python3.11-devel
gcc
procps-ng
vim
nano
```

To launch the build just run this command that will produce a container image that you can push to your OpenShift registry and/or any other registries you are leveraging for your PoC (like [Quay.io](https://quay.io)).
```
# ansible-builder build -v3 -t aap25-ee-vmware-rhel8:latest
```

### Important Note:
At the moment the current `vmware.community` collection has a bug. Using module `vmware_guest_disk` to change `disk_mode` has no effect. -> [community.vmware.vmware_guest_disk: changing "disk_mode" has no effect #2096
](https://github.com/ansible-collections/community.vmware/issues/2096).

For this reason I've made some changes in the code a submitted a [Pull Request for fixing this issue](https://github.com/ansible-collections/community.vmware/pull/2271).

While waiting the fix to be accepted and included in the main repository code, I've build a pre-release version based on the main branch of the repo including my fix. You will see that we are leveraging this version on the `requirements.yml` file.

## How to use the Ansible Execution Environment on OpenShift
In the `openshift` folder you will find two templates for your convenience to start the container with a dummy command that will keep the container running forever. After that you can jump on the container as you like, with `oc rsh` or via web terminal to leverage the example playbooks with the provided collections.

The files available on the `playbooks` directory will be copied in the EE container image under the path `/runner/project`.

All the playbooks accept parameters so you can launch them without changing the Ansible code, plus you may also instruct OpenShift in to launching it with the correct parameters. You can start from the `pod` or `job` template overriding the container's default start command.

### Usage Example
You could launch the playbook for reconfiguring all the disks of virtual machine to mode `persistent` launching the following command from the running container:

```
$ cd /runner/project
$ ansible-playbook -e vcenter_hostname=10.10.10.10 -e vcenter_username=administrator@vsphere.local -e vcenter_password='MySecretPassword' -e vcenter_dc=MYVMWDC -e vm_name=vm-test-migration-1 playbook-reconfigure-all-diskmode-persistent.yaml
```