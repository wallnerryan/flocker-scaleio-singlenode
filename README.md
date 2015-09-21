Quick Start for connecting Flocker to EMC ScaleIO
-------------------------------------------------

## Pre-requisits 

You must already have a ScaleIO Cluster Installed.
- See: https://github.com/wallnerryan/scaleio-vagrant-7

### Getting Started with this repository
```
git clone https://github.com/wallnerryan/flocker-scaleio-singlenode
cd flocker-scaleio-singlenode
```
### You can substitute your dev branch, or other release source code.
(you can alternatively skip this step, and install flocker from packages, if not developing. Just go through these initial step to install from packaging after vagrant up/ vagrant ssh. https://docs.clusterhq.com/en/1.3.1/install/install-client.html#other-linux-distributions)
```
git clone http://github.com/clusterhq/flocker
cd flocker/
```

### Get the vagrant env ready.
```
cp ../flocker-Vagrantfile .
cp ../pkgs/numactl-libs-2.0.9-4.el7.x86_64.rpm .
cp ../pkgs/libaio-0.3.109-12.el7.x86_64.rpm .
cp ../pkgs/EMC-ScaleIO-sdc-1.31-243.0.el7.x86_64.rpm .
```

### Your ready to go.
```
vagrant up
vagrant ssh
cd /vagrant/
sudo su
rpm -i libaio-0.3.109-12.el7.x86_64.rpm
rpm -i numactl-libs-2.0.9-4.el7.x86_64.rpm 
rpm -i EMC-ScaleIO-sdc-1.32-403.2.el7.x86_64.rpm 
/usr/bin/emc/scaleio/drv_cfg --add_mdm --ip <mdm gateway IP>

#Example if using the scaleio-vagrant-7 repo.
/usr/bin/emc/scaleio/drv_cfg --add_mdm --ip 192.168.50.12
```

### Now you ready to install/configuer flocker.

(The flocker source code is located at ```/vagrant/``` inside the machine.
See (https://docs.clusterhq.com/en/latest/) to get started Installing/Configuring Flocker.


#### Example Conifugrations Commands for Single Node Flocker connected to EMC ScaleIO

##### Install
```
cd /vagrant/
(Install Flocker from Source)
yum -yy install epel-release openssl openssl-devel libffi-devel python-virtualenv libyaml clibyaml-devel
virtualenv venv
source venv/bin/activate
/vagrant/venv/bin/python setup.py install
```

##### Configure

```
mkdir /etc/flocker
cd /etc/flocker
/vagrant/venv/bin/flocker-ca initialize mycluster
/vagrant/venv/bin/flocker-ca create-control-certificate mycluster.localdomain
cp control-mycluster.localdomain.crt control-service.crt
cp control-mycluster.localdomain.key control-service.key
/vagrant/venv/bin/flocker-ca create-api-certificate vagrantuser
cp vagrantuser.crt user.crt
cp vagrantuser.key user.key
chmod 0700 /etc/flocker
chmod 0600 /etc/flocker/control-service.key
/vagrant/venv/bin/flocker-ca create-node-certificate
ls -1 . | egrep '[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?.crt' | xargs -I {} cp {} /etc/flocker/node.crt
ls -1 . | egrep '[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?-[A-Za-z0-9]*?.key' | xargs -I {} cp {} /etc/flocker/node.key
chmod 0600 /etc/flocker/node.key
```

##### Configure Backend for Scaleio
```
mkdir /opt/flocker
cd /opt/flocker/
git clone https://github.com/emccorp/scaleio-flocker-driver
cd scaleio-flocker-driver/
/vagrant/venv/bin/python setup.py install

touch /etc/flocker/agent.yml
cat <<EOT >> /etc/flocker/agent.yml
version: 1
control-service:
  hostname: "localhost.localdomain"
dataset:
  backend: "scaleio_flocker_driver"
  username: "admin"
  password: "Scaleio123"
  mdm: "192.168.50.12"
  protection_domain: "pdomain"
  storage_pool: "pool1"
  ssl: True
EOT

/vagrant/venv/bin/flocker-control --verbose > /tmp/control.log 2>&1 &
/vagrant/venv/bin/flocker-container-agent --verbose > /tmp/container.log 2>&1 &
/vagrant/venv/bin/flocker-dataset-agent --verbose > /tmp/data.log 2>&1 &
```

##### To install ```flocker-volumes``` cli perform the following 
```
virtualenv --python=/usr/bin/python2.7 /opt/flocker/flocker-tools
/opt/flocker/flocker-tools/bin/pip install git+https://github.com/clusterhq/unofficial-flocker-tools.git
cd /etc/flocker
```

##### Run commands
```
/opt/flocker/flocker-tools/bin/flocker-volumes --control-service localhost list-nodes
SERVER     ADDRESS   
7f719abb   127.0.0.1 
```
