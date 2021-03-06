# About

This module provision prometheus instances on kvm.

The server will fetch and continuously update its configuration files using the keys found under a given key prefix in an etcd server.

Note that after the initial fetching of the configuration files, the server does not dependend on etcd to run prometheus and is only dependent on etcd for continuously updating its configurations with future configuration changes.

# Supporting Projects

If you don't have a strategy in place to upload configuration files in etcd, consider using the following terraform resource in our etcd terraform provider: https://registry.terraform.io/providers/Ferlab-Ste-Justine/etcd/latest/docs/resources/synchronized_directory

Otherwise, the module uses the following project to update its configuration: https://github.com/Ferlab-Ste-Justine/configurations-auto-updater

# Dependencies

The server expect an etcd server running with the v3 api.

Secure communication with the etcd server (tls with either client certificate or username/password authentication) is expected also.

See the following project to setup a secure etcd cluster in kvm if you don't already have a solution: https://github.com/Ferlab-Ste-Justine/kvm-etcd-server

A **prometheus.yml** configuration key is expected at the root of the keys prefix containing your configurations. You can configure other supporting configuration files (such as rules) as you wish.

# Supported Networking

The module supports libvirt networks and macvtap (bridge mode).

# Usage

## Input

- **name**: Name of the vm
- **vcpus**: Number of vcpus to assign to the vm. Defaults to 2.
- **memory**: Amount of memory in MiB to assign to the vm. Defaults to 8192 (ie, 8 GiB).
- **volume_id**: Id of the image volume to attach to the vm. A recent version of ubuntu is recommended as this is what this module has been validated against.
- **libvirt_network**: Parameters to connect to a libvirt network if you opt for that instead of macvtap interfaces. In has the following keys:
  - **ip**: Ip of the vm.
  - **mac**: Mac address of the vm. If none is passed, a random one will be generated.
  - **network_id**: Id (ie, uuid) of the libvirt network to connect to.
- **macvtap_interfaces**: List of macvtap interfaces to connect the vm to if you opt for macvtap interfaces instead of a libvirt network. Each entry in the list is a map with the following keys:
  - **interface**: Host network interface that you plan to connect your macvtap interface with.
  - **prefix_length**: Length of the network prefix for the network the interface will be connected to. For a **192.168.1.0/24** for example, this would be 24.
  - **ip**: Ip associated with the macvtap interface. 
  - **mac**: Mac address associated with the macvtap interface
  - **gateway**: Ip of the network's gateway for the network the interface will be connected to.
  - **dns_servers**: Dns servers for the network the interface will be connected to. If there aren't dns servers setup for the network your vm will connect to, the ip of external dns servers accessible accessible from the network will work as well.
- **cloud_init_volume_pool**: Name of the volume pool that will contain the cloud-init volume of the vm.
- **cloud_init_volume_name**: Name of the cloud-init volume that will be generated by the module for your vm. If left empty, it will default to ```<name>-cloud-init.iso```.
- **ssh_admin_user**: Username of the default sudo user in the image. Defaults to **ubuntu**.
- **admin_user_password**: Optional password for the default sudo user of the image. Note that this will not enable ssh password connections, but it will allow you to log into the vm from the host using the **virsh console** command.
- **ssh_admin_public_key**: Public part of the ssh key the admin will be able to login as
- **etcd**: Parameters to connect to the etcd backend. It has the following keys:
  - **ca_certificate**: Tls ca certificate that will be used to validate the authenticity of the etcd cluster
  - **etcd_key_prefix**: Prefix for all the domain keys. The server will look for keys with this prefix and will remove this prefix from the key's name to get the domain.
  - **etcd_endpoints**: A list of endpoints for the etcd servers, each entry taking the ```<ip>:<port>``` format
  - **client**: Authentication parameters for the client (either certificate or username/password authentication are support). It has the following keys:
    - **certificate**: Client certificate if certificate authentication is used.
    - **key**: Client key if certificate authentication is used.
    - **username**: Client username if certificate authentication is used.
    - **password**: Client password if certificate authentication is used.
- **prometheus**: Parameters to customise the behavior of prometheus. It has the following keys:
  - **web**: Object containing the following keys:
    - **external_url**: Value for the **--web.external-url** prometheus command line parameter. Has to be defined.
    - **max_connections**: Value for the **--web.max-connections** prometheus command line parameter. Set to 0 to use the default value (512).
    - **read_timeout**: Value for the **--web.read-timeout** prometheus command line parameter. Set to the empty string to use the default value (5m).
  - **retention**: Object containing the following keys:
    - **time**: Value for the **--storage.tsdb.retention.time** prometheus command line parameter. Set to the empty string to use the default value (15d).
    - **size**: Value for the **--storage.tsdb.retention.size** prometheus command line parameter. Set to the empty string to use the default value (0 for unlimited size).
- **chrony**: Optional chrony configuration for when you need a more fine-grained ntp setup on your vm. It is an object with the following fields:
  - **enabled**: If set the false (the default), chrony will not be installed and the vm ntp settings will be left to default.
  - **servers**: List of ntp servers to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#server)
  - **pools**: A list of ntp server pools to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#pool)
  - **makestep**: An object containing remedial instructions if the clock of the vm is significantly out of sync at startup. It is an object containing two properties, **threshold** and **limit** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#makestep)
- **fluentd**: Optional fluend configuration to securely route logs to a fluend node using the forward plugin. It has the following keys:
  - **enabled**: If set the false (the default), fluentd will not be installed.
  - **prometheus_tag**: Tag to assign to logs coming from prometheus
  - **configurations_updater_tag**: Tag to assign to logs coming from the prometheus configurations updater
  - **node_exporter_tag** Tag to assign to logs coming from the prometheus node exporter
  - **forward**: Configuration for the forward plugin that will talk to the external fluend node. It has the following keys:
    - **domain**: Ip or domain name of the remote fluend node.
    - **port**: Port the remote fluend node listens on
    - **hostname**: Unique hostname identifier for the vm
    - **shared_key**: Secret shared key with the remote fluentd node to authentify the client
    - **ca_cert**: CA certificate that signed the remote fluentd node's server certificate (used to authentify it)
  - **buffer**: Configuration for the buffering of outgoing fluentd traffic
    - **customized**: Set to false to use the default buffering configurations. If you wish to customize it, set this to true.
    - **custom_value**: Custom buffering configuration to provide that will override the default one. Should be valid fluentd configuration syntax, including the opening and closing ```<buffer>``` tags.