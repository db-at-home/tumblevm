# Tumble VM

A Vagrant VirtualBox VM of openSUSE Tumbleweed with 'ansible_local' provisioning

## Host Dependencies


Install Vagrant and Virtualbox on your host in your prefered manner. I used the pre-packaged binary bundles from each vendor below.

I've got this working on macOS and Linux platforms.  I also had vagrant working with libvirt on linux but this solution allows host platform flexibility.

### Working deploy of Oracle VirtualBox

At the time of this writing it is at 7.0.8 and [available here](https://www.virtualbox.org/wiki/Downloads).

You should only need to download and install the main 'platform package'

### Working depoy of Vagrant by HashiCorp

At the time of this writing it is at v2.3.4 and [available here](https://developer.hashicorp.com/vagrant/downloads).

You don't need the 'VMware Utility' as we're using VirtualBox.


## Vagrant Boxes

We are going to use [opensuse/Tumbleweed.x86_64](https://app.vagrantup.com/opensuse/boxes/Tumbleweed.x86_64) box from [Vagrant Cloud]( https://app.vagrantup.com/boxes/search).

In general I prefer openSUSE because of:

* zypper package management (which dnf is based on)
* yast2 system configuration (with console and graphical UI)
* QA'd rolling release process to deliver the latest _working_ and _integrated_ releases.
* availability of the [openSUSE Build Service](https://build.opensuse.org/) repos and package and image building tools.

This last one is really big, because with a tool like [openSUSE KIWI](https://osinside.github.io/kiwi/) in the Build Service, you can build your own custom Vagrant boxes [like these](https://build.opensuse.org/package/show/openSUSE:Factory:ToTest/kiwi-images-vagrant).

## Test Host VM Handling

In a shell run the following to exercise all the parts and make sure things are in working order.

    mkdir testbox
    cd testbox
    vagrant init -m opensuse/Tumbleweed.x86_64
    vagrant up --provider virtualbox
    vagrant ssh
    cat /etc/os-release # see the guest VM is working
    exit # exit the guest
    vagrant halt
    vagrant destroy --force
    cd ..
    rm -rf testbox

## Knobs for the Guest VM with Provisioning

The whole point of this project is to use [ansible_local](https://developer.hashicorp.com/vagrant/docs/provisioning/ansible_local) provisioning on the VM and make it more 'home network lab' friendly and accessible to a 'developer' and allow for further in-VM ansible runs to rapidly host complex development environments with different goals. Like a heavy-weight docker that is agnostic of the host system it is running on and always allows you to use a familiar linux environment.

***NOTE***: This is NOT a secured VM suitable for deployment in *any* non-private network environment. Do not deploy your network services using this method. This is [echoing a warning](https://developer.hashicorp.com/vagrant/docs/networking/public_network) from HashiCorp about the unsafe nature of the `public_network` option.

### Vagrantfile

You can change `config.vm.box` as you like, but the provisioning expects 'zypper' for package and pattern/group install of deps.

The `config.vm.network "public_network"` is to bridge the VM to a physical network device on the host, and make the VM look like it is hardware sitting on the same network as the host. This allows for development and testing without port-mapping for web or other services. You can avoid an interactive promot when bringing the VM up by specifying the string of the bridge device to use in the Vagrantfile:

      config.vm.network "public_network", ip: "192.168.88.5", bridge: [
        "en6: USB 10/100/1000 LAN",
        "en0: Wi-Fi"
      ]

***NOTE***: If your host network interface is on a network that does not have a DHCP server then you will need to include the `ip` and modify the value `, ip: "192.168.88.5"` to be on the same network as that of your host.

Adjust `vb.cpus` and `vb.memory` to meet your VM perf requirements.

Finally, `config.vm.provision "ansible_local"` has the settings that allow Vagrant to [provision the VM]() on `vagrant up` and every subsequent invocation of `vagrant up --provision` and make use of Ansible and its logic to make changes only if needed.

### hosts

The ansible inventory file for the guest VM has two vars used to access the guest VM:

    vguser = flax      # username for ssh -l <vguser> ...
    vghost = tumblevm  # hostname for ssh ... <vghost>.local

## Launch

After adjusting the Vagrant and host files as desired we bring up the guest, similar to the earlier install test. Now more simply do:

    vagrant up

And when prompted provide the 'digit(s)' that index the physical interface you want the VM to bridge to.

    ==> default: Available bridged network interfaces:
    ...
        default: Which interface should the network bridge to?

This will bring up the VM, the bridge interface as `eth1` (unless you include a `private_network` in your Vagrantfile), mount the running dir under `/vagrant` in the VM, install ansible in the VM and the galaxy collection specified, and then run the `ansible_local` provision.yml playbook.

If any issues occur that produce `FAILED!` ansible output, address the issues and rerun the provisioning (without shutting down the VM) using:

    vagrant up --provision

If the error was with vagrant before it got to the provisioning step do `vagrant halt` before the `vagrant up --provision` command.

## Connect

Once the VM is up you should be able to ping it by your `vghost` name specified in the `hosts` file. For example:

    {16:18}~/git/tumblevm:main ✗ ➭ ping -c3 tumblevm.local
    PING tumblevm.local (192.168.88.5): 56 data bytes
    64 bytes from 192.168.88.5: icmp_seq=0 ttl=64 time=1.127 ms
    64 bytes from 192.168.88.5: icmp_seq=1 ttl=64 time=0.950 ms
    64 bytes from 192.168.88.5: icmp_seq=2 ttl=64 time=0.522 ms
    
    --- tumblevm.local ping statistics ---
    3 packets transmitted, 3 packets received, 0.0% packet loss
    round-trip min/avg/max/stddev = 0.522/0.866/1.127/0.254 ms

With this you can ssh to your guest over the bridged network with something similar to the following, adjusted for any `vgname` or `vghost` modifications you have made in the `hosts` file:

    {16:21}~/git/tumblevm:main ✗ ➭ ssh -l flax tumblevm.local
    Have a lot of fun...
    flax@tumblevm:~> grep PRETTY /etc/os-release
    PRETTY_NAME="openSUSE Tumbleweed"
    flax@tumblevm:~> exit
    logout
    {16:24}~/git/tumblevm:main ✗ ➭



## Additional Ansible

For running additional playbooks in the VM use `vagrant ssh` to log on to the guest and cd to `/vagrant` and this allows for the `vagrant` user to run ansible playbooks in the VM the same way that the `ansible_local` provisioning step did.

For example:

    {16:21}~/git/tumblevm:main ✗ ➭ vagrant ssh
    Have a lot of fun...
    vagrant@tumblevm:~> cd /vagrant
    vagrant@tumblevm:/vagrant> ansible-playbook devel_basis.yml

    PLAY [Install devel_basis pattern]*****************************

    TASK [Install pattern(s) with recommends] *********************
    changed: [tumblevm]

    PLAY RECAP ****************************************************
    ...

Will install git, c/c++ compliers, and other devel tools in the guest.

