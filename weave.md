
# Install Container Networking Plugins

Overview: The linux box (Fedora/CoreOS/RHEL) may be missing part of the required container network plugins. Or you may be like me and have accidentally nuked them from your machine. Below is an easy way to reinstall the missing plugins. 

```shell

mkdir -p /tmp/cni-plugins

cd /tmp/cni-plugins

wget https://github.com/containernetworking/plugins/releases/download/v1.0.0/cni-plugins-linux-amd64-v1.0.0.tgz

tar xfvz cni-plugins-linux-amd64-v1.0.0.tgz

# Copy the items needed for your CNI. Weave uses the two below
sudo cp {loopback,portmap} /opt/cni/bin/

```

