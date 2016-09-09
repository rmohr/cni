# libkv plugin

## Overview

This is a "meta-plugin". It reads in its own netconf. According to the conf
it loads a JSON array of types.NetConf from the specified key/value store.
Then it delegates one loaded NetConf after the other to the specified plugin.
This allows storing the whole CNI configuration in a remote place. The first
NetConf in the array will be treated as the main configuration and it's
configuration will be returned as result to the caller.

The plugin searches for the configuration on the location
<basePath>/<containerId> in the store. This allows different configurations
per container.


## Operation

Make sure to have either `consul`, `etcd` or `zookeeper` running. The
following scenario assumes `etcd` at `localhost:2379`.

First store the cni configuration of the libkv plugin in the standard
cni config folder:

```bash
cat >/etc/cni/net.d/10-mylibkv.conf <<EOF
{
  "name": "libkv",
  "type": "libkv",
  "storeBackend": "etcd",
  "uri": "localhost:2379",
  "basePath": "cni/",
  "storeConfig": {
    "connectionTimeout": 10
  }
}
EOF
```

Then store the actual configuration for the container `testcontainer` as json array in etcd:

```bash
etcdctl set cni/testcontainer <<EOF
[
  {
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni2",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
      "type": "host-local",
      "subnet": "10.23.0.0/16",
      "routes": [
        {
          "dst": "0.0.0.0/0"
        }
      ]
    }
  },
  {
    "type": "loopback"
  }
]
```

Then create and configure the network namespace `testcontainer`:

```bash
netnspath=/var/run/netns/testcontainer

ip netns add $contid
./exec-plugins.sh add testcontainer $netnspath
```

Look around in the container:

```bash
ip netns exec testcontainer ip a
```

Let cni do some cleanup and delete the namespace:

```bash
./exec-plugins.sh del testcontainer $netnspath
ip netns delete testcontainer
```

## Example Configuration

```json
{
  "name": "libkv",
  "type": "libkv",
  "storeBackend": "etcd",
  "uri": "localhost:2379",
  "basePath": "cni/",
  "storeConfig": {
    "connectionTimeout": 10
  }
}
```

## Network configuration reference

* `name` (string, required): the name of the network.
* `type` (string, required): "libkv".
* `storeBackend` (string, required): name of the backend. Either `consul`, `etcd` or `zookeeper`.
* `uri` (string, required): Store connection URI. 
* `basePath` (string, optional): Base path where to search for the key.
* `storeConfig` (dictionary, optional): Additional store connection options (credentials, tls, ...).

