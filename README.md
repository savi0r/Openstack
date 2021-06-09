<h2>Openstack:</h2>

so what is openstack?
OpenStack is a one stop for all, it can manage the whole infrastructure at once. It is officially explained like this  
> a cloud operating system that controls large pools of compute, storage, and networking resources throughout a datacenter, all managed and provisioned through APIs with common authentication mechanisms.

Because openstack has a huge number of services , for the sake of simplicity we just introduce the core components:
* Keystone (identity service) : it is more like a security guard in a company which provieds you an access card - Token ID - which based on your role authorizes you to enter different parts of the company.
* Nova (compute service) : this is more like an IT hardware allocation team , Upon successful verification, the IT hardware allocation team will approve his desktop workstation request for the required configuration (Ubuntu OS/8GB RAM/500GB HDD) and forward the request to the desktop assembly unit subdivision. The desktop assembly team will assemble the new desktop machine with the requested configuration of 8GB RAM and a 500GB hard disk.
* Glance (image service) : it is more like an image management team - if such thing exist :D- The image management team will act as an ISO image repository for the IT hardware allocation team and will provide the selected OS for the IT hardware allocation team to install the operating system in the assembled bare metal.
* Neutron (networking service) : this one is more like network team , it takes care of all networking related activities.

<h2>Openstack installation:</h2>

If you've ever tried to take a look at openstack modular architechture , you might have some idea about how perplexing it is to know it's different building blocks set aside installing them all together because it is so hard to install every other parts of it manually there are some different solutions for that and we chose kolla ansible because it stood the test of time.

As you may came across ansible has two types of installation :
* All-in-one : which basically is running all services on one node , and I think it is obvious that it is not useful for production environment but we use it here as a showcase
* Multinode : you just spread your services between different nodes and it provides more resources and also failover functionality.

I ran my environment on Centos8 and you can find my resources <a href=https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html>here</a>.

You also need to meet these requirements:

2 network interfaces

8GB main memory # in my case of experience 8 GB was not enough and I had to go with 12 GB , in a multi node environment that would be the case for compute nodes

40GB disk space


Set the DNS to use Shecan:

```
vi /etc/resolve.conf
nameserver 178.22.122.100
```

There is also a better solution rather than Shecan which will spare you from a lot suffering which you can find out more about in <a href=https://github.com/freedomofdevelopers/fod>here </a>

Install Python build dependencies:

```
sudo dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux
```

Ensure the latest version of pip is installed:

```
pip3 install -U pip
```

Install Ansible. Kolla Ansible requires at least Ansible 2.9 and supports up to 2.10:

```
pip3 install 'ansible<2.10'
```

Install kolla-ansible and its dependencies using pip:

```
sudo pip3 install kolla-ansible --ignore-installed PyYAML
```

Create the /etc/kolla directory:

```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy globals.yml and passwords.yml to /etc/kolla directory:

```
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

Copy all-in-one and multinode inventory files to the current directory:

```
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```

For best results, Ansible configuration should be tuned for your environment. For example, add the following options to the Ansible configuration file `/etc/ansible/ansible.cfg`:

```
mkdir /etc/ansible
vi /etc/ansible/ansible.cfg
```
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

Passwords used in our deployment are stored in /etc/kolla/passwords.yml file. All passwords are blank in this file and have to be filled either manually or by running random password generator:

```
kolla-genpwd
```

/etc/kolla/globals.yml is the main configuration file for Kolla Ansible. There are a few options that are required to deploy Kolla Ansible:

```
kolla_base_distro: "centos"
kolla_install_type: "source"
First interface to set is “network_interface”. This is the default interface for multiple management-type networks:
network_interface: "eth1"
#Second interface required is dedicated for Neutron external (or public) networks, can be vlan or flat, depends on how the networks are created:
neutron_external_interface: "eth2"
#floating IP for management traffic. This IP will be managed by keepalived to provide high availability, and should be set to be not used address in management network that is connected to our network_interface:
kolla_internal_vip_address: "192.168.10.100"

```
```
kolla-ansible -i ./all-in-one bootstrap-servers
You may need to login to your docker account in order to not hit any errors in the way 

kolla-ansible -i ./all-in-one prechecks

kolla-ansible -i ./all-in-one deploy #before deployment in order to make the deploy faster and maybe error free you can pull docker images via this command kolla-ansible -i all-in-one pull
```

Install the OpenStack CLI client:

```
pip install python3-openstackclient
```

OpenStack requires an openrc file where credentials for admin user are set. To generate this file:

```
kolla-ansible post-deploy
```

provide cli with admin credentials using :
Before we start using the OpenStack command tool, we need to provide OSC with information on your OpenStack username, password, and project, as well as an endpoint URL for KeyStone to contact for authentication and getting the list of endpoint URLs for the other OpenStack service.

```
. /etc/kolla/admin-openrc.sh
```

the file looks something like this :

```
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=GJHhDEn2GpZf181gy8EmRfCdABWzzgtETbycOHVy
export OS_AUTH_URL=http://192.168.1.100:35357/v3
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE=internalURL
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
```

You can change this file accordingly and make new openrc files for your other users

To verify if the environment
variables are set correctly, execute the following command:

```
export | grep OS_
```

Depending on how you installed Kolla Ansible, there is a script that will create example networks, images, and so on.

For deployment or evaluation, run init-runonce script:

```
/usr/local/share/kolla-ansible/init-runonce
```

<hr>
<h2>Openstack administration:</h2>
Now we're going to deploy an instance step by step, so let's move along.

You can list images using :

```
openstack image list
```

You can also add new images by first downloading the image file and then execute the following command:

```
wget https://cloud-images.ubuntu.com/releases/14.04.4/release-20170831/ubuntu-14.04-server-cloudimg-amd64-disk1.img

openstack image create "Ubuntu14_04" \
--file ubuntu-14.04-server-cloudimg-amd64-disk1.img \
--disk-format qcow2 --container-format bare
```


List all available flavors in this project:

```
openstack flavor list
#you can choose your prefered flavor for instance creation
```

generate a new key pair:

```
ssh-keygen -q -N ""
openstack keypair create --public-key ~/.ssh/id_rsa.pub keypair
```

By default, the security group named default applies to all instances that have firewall rules that deny remote access to the instances. So, we need to append the new rule to the security group to allow SSH connection to the VM:

```
openstack security group rule create --proto tcp --dst-port 22 default
```

check the created rule by the following command:

```
openstack security group show default
```

List available networks for the project:

```
openstack network list
#you can grab the id of the network from here for the instance creation
```

For deploying an instance:

```
openstack server create --flavor m1.small \
--image Ubuntu14_04  \
--nic net-id=public \
--security-group default \
--key-name keypair instance1


openstack server create --flavor m1.tiny \
--image cirros  \
--nic net-id=public \
--security-group default \
--key-name keypair instance2
```

You can see the status of your instance that you've just created using the following command:

```
openstack server list
#if the status became "active" then we're all up and good
```

Connecting to the instance using SSH:
To connect the instance from the external network, we need to associate the floating IP from the external network
Create a floating IP address on the virtual provider network using the following command:

```
openstack floating ip create public
```

#add the floating_ip_address value from the above command output to the instance

```
openstack server add floating ip [openstack server ID | openstack server name ] [floating_ip_address]
ex:
openstack server add floating ip 6cd5eaa3-7249-45ea-97b8-2941cbdf1d57 10.0.2.172
```

Now you can ssh to the instance by using that floating ip address value
ex:
`ssh ubuntu@10.0.2.172`


Tip: If there are some problems with libvirt because libvirt cannot emulate hardware in Virtual environments using KVM instead of that you should go with qemu which is the case in Arvan cloud instances.

```
mkdir -p /etc/kolla/config
vi /etc/kolla/config/nova.conf
```
```
#add below contents
[libvirt]
virt_type=qemu
cpu_mode = none
```

After adding your custom configs will need:

```
kolla-ansible reconfigure # (Apply change and restart required containers)
kolla-ansible genconfig # and restart manually the containers
```

for more information check <a href=https://docs.openstack.org/kolla-ansible/rocky/admin/deployment-philosophy.html>here</a>

If there were any connectivity problems make sure that selinux is in permissive mode and ip forwarding is enabled

```
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

For further information check <a href=https://thenewstack.io/solving-a-common-beginners-problem-when-pinging-from-an-openstack-instance>here</a>



<h3>Integrating ceph and openstack:</h3>
I just integrated ceph as a storage for glance images but the case is the same for other storage services such as cinder and etc.

For integrating glance with ceph change these settings in `/etc/kolla/globals.yml`

```
external_ceph_cephx_enabled: "yes"
ceph_glance_keyring: "ceph.client.glance.keyring"
ceph_glance_user: "glance"
ceph_glance_pool_name: "images"
glance_backend_ceph: "no"
glance_backend_file: "no"
```

Then add the specified node to the ceph-ansible inventory as a client and do copy ssh-copy-id on ceph master node and then run , remember to edit `/etc/hosts`

```
ansible-playbook -i hosts site.yml --limit [hostname]
ex:
ansible-playbook -i hosts site.yml site.yml --limit kolla
```

with the above commands you must have the ceph-common packages on the openstack node , you also have ceph.conf in `/etc/ceph/ceph.conf` now we must create a pool and a user on cephmon and copy that to openstack node

```
ceph osd pool create images 128

ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'

ceph auth get-or-create client.glance |
ssh os-node1 sudo tee /etc/ceph/ceph.client.glance.keyring

```

Log in to openstack node and run the following commands:

```
ceph -s --id glance
```

I did also another step and attached my key to the end of ceph.conf, so I do not have to provide my keyring every now and then.

```
[client.glance]
keyring = /etc/ceph/ceph.client.glance.keyring
```

if everything was okay then copy /etc/ceph/* to /etc/kolla/config/glance/ afterwards run :

```
kolla-ansible -i all-in-one deploy
```

for further check download a new image and add it to the glance service

```
wget http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-
x86_64-disk.img

openstack image create "cirros_test" \
--file cirros-0.3.1-x86_64-disk.img \
--disk-format=qcow2 --container-format=bare

```

You can verify that the new image is stored in Ceph by querying the image ID in the Ceph images pool:

```
rbd -p images ls --id glance
rbd info images/<image name> --id glance
```

