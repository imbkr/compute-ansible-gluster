## This is not an official Google product

# Introduction

This tutorial will take you through the provisioning and testing of a multinode
GlusterFS cluster on GCE.

[![asciicast](https://asciinema.org/a/33428.png)](https://asciinema.org/a/33428)

# Pre-requisites

First you will need to download some credentials and set some environment
variables in order to authenticate with GCP. Follow [this
tutorial](http://docs.ansible.com/ansible/guide_gce.html#credentials) to get
your JSON credentials file.

Once you have your JSON credentials file on your local machine set the following environment
variables that will be used by Ansible's GCP modules in order to create
resources:

    export GCE_EMAIL=<your-service-account-email>
    export GCE_PROJECT=<your-project-id>
    export GCE_CREDENTIALS_FILE_PATH=<path-to-your-newly-created-pem-file>

You will also need to ensure that you have your local machine's SSH key uploaded
to your project's metadata. For more information read our guide on [adding and removing SSH keys](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys?hl=en).

In order to run the playbook you will need to install Ansible and Libcloud as
follows:

    pip install ansible "apache-libcloud>=0.19.0"

Once those dependencies have been installed, clone the repository and enter the
directory:

    git clone https://github.com/GoogleCloudPlatform/ansible-gluster
    cd ansible-gluster

# Cluster Configuration

In the repository you will find a `gluster.yml` file that contains parameters at
the top for configuring your cluster.

- `machine_type`: The size of your each machine in the cluster, more cores will give you
  better network performance. The full list of machine types can be found
  [here](https://cloud.google.com/compute/docs/machine-types).
- `hosts`: This list defines both the names of your host and your total cluster
  size
- `disk_type`: The type of disk to use for Gluster's data volume. This can be
  either pd-ssd or pd-standard.
- `disk_size`: The size of the data disk that will be attached to each instance
  in the cluster. Keep in mind that the IOPS and throughput performance of each
  disk scales with its size. More info on persistent disk performance can be
  found [here](https://cloud.google.com/compute/docs/disks/#pdperformance).
- `zone`: This defines the specific zone with a region that the cluster will be
  provisioned in.

# Gluster volume configuration

In order to create Gluster volumes at provisioning time you can define the
`volumes` list variable. Each element in the list is a dictionary with the
following keys that describe the type of volume to create in the cluster.

- `name`: Identifier for your volume
- `type`: Gluster volume type. Examples are `stripe 3`, `replica 3`, or `stripe 3 replica 3`. Can also be left blank for a distributed volume (default).
- `parameters`: List of parameters for the volume.
  [Full
  list](http://www.gluster.org/community/documentation/index.php/Gluster_3.2:_Setting_Volume_Options)
- `hosts`: The list of hosts that this volume will be created on.

Volumes will be mounted at /mnt/NAME by default. The mount path can be changed
by setting the global variable `mount_point`.

# Deploying

Once you have configured your cluster specifications you can provision it by
running the Ansible playbook as follows:

    export ANSIBLE_HOST_KEY_CHECKING=False
    ansible-playbook -i hosts gluster.yml
    cat /tmp/gluster-client-*/*/tmp/* # view your results

# Testing your setup

In order to mount your Gluster volume, launch an instance in your network and
then run the following:

    sudo yum install glusterfs-client
    sudo mount -t glusterfs <ip-of-a-machine-in-your-cluster>:/share /mnt

For a quick example, you can run the gluster-clients.yml playbook which will take
care of the provisioning a client for you and run a benchmark with
[fio](https://github.com/axboe/fio).

    export ANSIBLE_HOST_KEY_CHECKING=False
    ansible-playbook -i hosts gluster-clients.yml

# Monitoring and logging

These playbooks also setup the Stackdriver and Logging agents on the instances.
You can checkout your logs [here](https://console.developers.google.com/logs) and your monitoring
[here](https://app.google.stackdriver.com/).

# Destroying your cluster and clients

You can use the Ansible playbooks to turn down your machines by running the
following:

    ansible-playbook -i hosts gluster.yml gluster-clients.yml -e state=absent
