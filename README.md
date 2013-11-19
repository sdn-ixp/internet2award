### Installation steps:

* Download the Internet2-SDX virtual machine (VM) image, below, and you are all set :)

``` sh
$ wget http://sites.noise.gatech.edu/~shahbaz/internet2-sdx
```

The username and password for the VM are 'sdx'.

#### Miscellaneous 

1. Follow the instructions, [here](https://docs.google.com/document/d/1j4juaVb0TVKI2Ckc1Q5PEgiN3lcHeBRMeve1cU-Zdz4/edit#heading=h.17i67qvy3ght), to setup the VirtualBox and enable SSH access on your VM. 

2. Visit the following sites to learn about Pyretic, POX, and Mininet:

* [Pyretic](http://frenetic-lang.org/pyretic/)
* [POX](http://www.noxrepo.org/pox/about-pox/)
* [Mininet](http://mininet.org)

### SDX Platform

The SDX platform runs as a module on top of the Pyretic runtime. It consists of `main.py` file, which reads the `sdx_global.cfg` and `sdx_policies.cfg` configuration files. The `sdx_global.cfg` points to the topology file e.g., `topology/mininet.topo`, that contains information about the overall topology of the IXP i.e., how many autonomous systems (ASes) are connected, which ports they are connected to, and who they are peering with at the IXP. Whereas, the `sdx_policies.cfg` lists the active policies for each participant which will be composed together, processed and applied to each incoming packet. Here's an example configuration:

* sdx_global.cfg
``` json
["topology/mininet.topo"]
```

* topology/mininet.topo
``` json
{
        "A": {"Ports": [{"Id": 1, "MAC": "00:00:00:00:00:01"}],
              "Peers": ["B"]},  
        "B": {"Ports": [{"Id": 2, "MAC": "00:00:00:00:00:02"}],
              "Peers": ["A", "C"]},
        "C": {"Ports": [{"Id": 3, "MAC": "00:00:00:00:00:03"},
                        {"Id": 4, "MAC": "00:00:00:00:00:04"}],
              "Peers": ["B"]}
}
```

* sdx_policies.cfg
``` json
{
        "A": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_A"],
        "B": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_B"],
        "C": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_C"]
}
```

#### Policies

The policies are provided under the `policies` folder. Participants can write any type of policy using the language constructs provided in Pyretic. Each participant writes policies in its own python script which reads the announced prefixes from the accompanying `local.cfg` file at run time. Following is an example of the `traffic offloading` policy:

* python policy
``` python
...
def policy(participant, fwd):
    '''
        Specify participant policy
    '''
    participants = parse_config(cwd + "/pyretic/sdx/policies/traffic_offloading_ip_prefixes/local.cfg")
    
    return (
        (parallel([match(dstip=participants["A"]["IPP"][i]) for i in range(len(participants["A"]["IPP"]))]) 
          >> modify(srcmac=participant.phys_ports[0].mac) >> fwd(participant.peers['A'])) +
        (parallel([match(dstip=participants["B"]["IPP"][i]) for i in range(len(participants["B"]["IPP"]))]) 
          >> fwd(participant.phys_ports[0])) + 
        (parallel([match(dstip=participants["C"]["IPP"][i]) for i in range(len(participants["C"]["IPP"]))]) 
          >> modify(srcmac=participant.phys_ports[0].mac) >> fwd(participant.peers['C']))
    )
...
```

* local.cfg
``` json
{
        "A": {"IPP": ["110.0.0.0/16"]},
        "B": {"IPP": ["120.0.0.0/16"]},
        "C": {"IPP": ["130.0.0.0/16"]}
}
```

### Mininet Topologies

We use mininet, as a rapid prototyping and development platform, for building and testing the applications written atop SDX Platform. For each policy, listed in the `policies/` folder, we provide an accompanying mininet script, under `scripts/` folder, that creates and configures a network according to the topology written in the `topology/mininet.topo` file. (At the moment, we have hardcoded the topology in the scripts. In later versions, we will provide an automated model for reading the topology information from the `topology/mininet.topo` and configuring the network accordingly). Once the network is setup, the script then generates test packets to perform functional testing tailored for the given policies.

* Example script (scripts/sdx_mininet_simple.py):

``` python
def simple(cli, controllerIP):
    "Create and test SDX Simple Module"
    print "Creating the topology with one IXP switch and three participating ASs\n\n" 
    topo = SingleSwitchTopo(k=3)
    net = Mininet(topo, controller=lambda name: RemoteController( 'c0', controllerIP ), autoSetMacs=True)
    net.start()
    hosts=net.hosts
    print "Configuring participating ASs\n\n"
    for host in hosts:
        if host.name=='h1':
            host.cmd('ifconfig lo:40 110.0.0.1 netmask 255.255.255.0 up')
            host.cmd('route add -net 120.0.0.0 netmask 255.255.255.0 gw 10.0.0.2 h1-eth0')
            host.cmd('route add -net 130.0.0.0 netmask 255.255.255.0 gw 10.0.0.2 h1-eth0')
        if host.name=='h2':
            host.cmd('route add -net 110.0.0.0 netmask 255.255.255.0 gw 10.0.0.1 h2-eth0')
            host.cmd('ifconfig lo:40 120.0.0.1 netmask 255.255.255.0 up')
            host.cmd('route add -net 130.0.0.0 netmask 255.255.255.0 gw 10.0.0.3 h2-eth0')
        if host.name=='h3':
            host.cmd('route add -net 110.0.0.0 netmask 255.255.255.0 gw 10.0.0.2 h3-eth0')
            host.cmd('route add -net 120.0.0.0 netmask 255.255.255.0 gw 10.0.0.2 h3-eth0')
            host.cmd('ifconfig lo:40 130.0.0.1 netmask 255.255.255.0 up')
    if (cli): # Running CLI
        CLI(net)
    else:
        print "Running the Ping Tests\n\n"
        for host in hosts:
            if host.name=='h1':
                host.cmdPrint('ping -c 5 -I 110.0.0.1 130.0.0.1')

    net.stop()
    print "\n\nExperiment Complete !\n\n"
```

### Examples

We have provided three examples in the code repository: (1) A simple policy, (2), traffic-offloading policy, and (3) inbound-TE policy. Each has two flavors, one is using only the IP addresses and the other one using IP prefixes (this is a new feature provided in the latest release of Pyretic).

Each of these examples, assume three participants, A, B and C; all having a peering relationship with each other. A and B connect to only one port at the IXP, while C connects at two ports namely (C1, and C2). We implement a `single` topology in mininet, where the three participants are connecting to a single switch.

Here, we will show the steps needed to run the SDX platform using two examples:

#### 1. Simple

In simple policy, we only enable connectivity between A and B and block all communication with C. Note, that C still maintains a peering relationship with A and B but the data-plane policy enforced by the SDX platform will not allow any data traffic to passthrough from A and B to C. Here're the steps for running the simple policy:

1. Make sure that `sdx_global.cfg` has the following content:

``` json
["topology/mininet.topo"]
```

2. Change the `sdx_policies.cfg` to have the following:

``` json
{
        "A": ["pyretic.sdx.policies.simple.participant_A"],
        "B": ["pyretic.sdx.policies.simple.participant_B"],
        "C": ["pyretic.sdx.policies.simple.participant_C"]
}
```

3. Run SDX platform

``` sh
$ cd ~/pyretic
$ ./pyretic.py pyretic.sdx.main
```

4. In an other terminal, run the `sdx_mininet_simple.py` script:

``` sh
$ cd ~/pyretic/pyretic/sdx/scripts
$ sudo sdx_mininet_simple.py
```

Once running you should see that participant A can ping participant B. To perform the ping test with C, run mininet in cli mode:

``` sh
$ sudo sdx_mininet_simple.py --cli
```

Then in the mininet prompt, run the following:

``` sh
mininet> ping -c 5 -I 110.0.0.1 130.0.0.1
```

The ping test will fail this time as there is no rule installed on the switch for packets going from A to C.

#### 2. Inbound Traffic-Engineering

In inbout-TE policy, we do traffic engineering on the traffic coming to C from A or B. We distribute the traffic based on the IP prefixes. In this example, all traffic coming for the IP prefix `130.0.0.0/16` will be routed to port C1 and for `"140.0.0.0/16` will be routed to C2. Here're the steps for running this policy:

1. Make sure that `sdx_global.cfg` has the following content:

``` json
["topology/mininet.topo"]
```

2. Change the `sdx_policies.cfg` to have the following:

``` json
{
        "A": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_A"],
        "B": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_B"],
        "C": ["pyretic.sdx.policies.inbound_traffic_engineering_ip_prefixes.participant_C"]
}
```

3. Run SDX platform

``` sh
$ cd ~/pyretic
$ ./pyretic.py pyretic.sdx.main
```

4. In an other terminal, run the `sdx_mininet_inbound_traffic_engineering.py` script:

``` sh
$ cd ~/pyretic/pyretic/sdx/scripts
$ sudo sdx_mininet_inbound_traffic_engineering.py
```

Once running you should see that the pings originating from C1, with source IP `130.0.0.1`, for A are passing but the ones with source IP `140.0.0.1` are not. Similarly, the opposite happens for C2. This is because the replies for the ping, with source IP `140.0.0.1`, from C1 are being sent to C2 based on the traffic-engineering rule applied by the SDX platform. 

## Contact Us

Please contact us at `muhammad.shahbaz@gatech.edu` for any questions or concerns. 
