# Working with a simple SDN controller

In this exercise, you will learn about an open-source OpenFlow controller `POX`. You will learn how to write network applications, i.e., Hub and Layer 2 MAC Learning etc., on POX and run them on a virtual network based on Mininet.

After the exercise, you will be asked to create and submit a network application that implements Layer 2 Firewall that disables inbound and outbound traffic between two systems based on their MAC address. 

## Overview

The network you'll use in this exercise includes 3 hosts and a switch with an OpenFlow controller (POX):

<!-- ![500pt](images/topo.png =250x)

<img src="images/topo.jpg" alt="topo" width="200"/> -->

<img src="https://github.com/wontoniii/simple-controller/blob/main/images/topo.png" width="350" />

Figure 1: Topology for the Network under Test

## Requirements

Before starting the exercise, make sure that your machine has the following two requirements installed: 

1. The POX controller
2. xterm

To install the POX controller, it is sufficient to close on your machine the POX repository in your folder of choice:

```bash
git clone http://github.com/noxrepo/pox
```

POX is a Python based SDN controller platform geared towards research and education. For more details on POX, see the [POX Documentation](https://noxrepo.github.io/pox-doc/html/) for more info.

To install xterm execute:

```bash
sudo apt install xterm
```

## Starting the pox controller

We’re not going to be using the reference controller anymore, which is the default controller that Mininet uses during its simulation. Make sure that it’s not running in the background:

```bash
$ ps -A | grep controller
```

If so, you should kill it either press Ctrl-C in the window running the controller program, or from the other SSH window:

```bash
$ sudo killall controller
```
You should also run `sudo mn -c` and restart Mininet to make sure that everything is clean and using the faster kernel switch: From you Mininet console:

```bash
mininet> exit  
$ sudo mn -c  
$ sudo mn --topo single,3 --mac --switch ovsk --controller remote
```

The POX controller comes pre-installed with the provided VM image.

Now, run the basic hub example:

```bash
$ pox.py log.level --DEBUG forwarding.hub
```

This tells POX to enable verbose logging and to start the hub component.

The switches may take a little bit of time to connect. When an OpenFlow switch loses its connection to a controller, it will generally increase the period between which it attempts to contact the controller, up to a maximum of 15 seconds. Since the OpenFlow switch has not connected yet, this delay may be anything between 0 and 15 seconds. If this is too long to wait, the switch can be configured to wait no more than N seconds using the --max-backoff parameter. Alternately, you exit Mininet to remove the switch(es), start the controller, and then start Mininet to immediately connect.

Wait until the application indicates that the OpenFlow switch has connected. When the switch connects, POX will print something like this:

```
INFO:openflow.of_01:[Con 1/1] Connected to 00-00-00-00-00-01  
DEBUG:samples.of_tutorial:Controlling [Con 1/1]
```

## Verify Hub behavior with tcpdump

Now verify that hosts can ping each other, and that all hosts see the exact same traffic - the behavior of a hub. To do this, we'll create xterms for each host and view the traffic in each. In the Mininet console, start up three xterms:

```bash
mininet> xterm h1 h2 h3
```

Arrange each xterm so that they're all on the screen at once. This may require reducing the height to fit a cramped laptop screen.

In the xterms for h2 and h3, run< ```tcpdump```, a utility to print the packets seen by a host:

```bash
tcpdump -XX -n -i h2-eth0
```
and respectively:

```bash
tcpdump -XX -n -i h3-eth0
```

In the xterm for h1, send a ping:

```bash
ping -c 1 10.0.0.2
```

The ping packets are now going up to the controller, which then floods them out all interfaces except the sending one. You should see identical ARP and ICMP packets corresponding to the ping in both xterms running tcpdump. This is how a hub works; it sends all packets to every port on the network.

Now, see what happens when a non-existent host doesn't reply. From h1 xterm:

```bash
ping -c 1 10.0.0.5
```

You should see three unanswered ARP requests in the tcpdump xterms. If your code is off later, three unanswered ARP requests is a signal that you might be accidentally dropping packets.


You can close the xterms now.

Now, lets look at the hub code:

```python
#!/usr/bin/python

from pox.core import core
import pox.openflow.libopenflow_01 as of
from pox.lib.util import dpidToStr
log = core.getLogger()
def _handle_ConnectionUp (event):
  msg = of.ofp_flow_mod()
  msg.actions.append(of.ofp_action_output(port = of.OFPP_FLOOD))
  event.connection.send(msg)
  log.info("Hubifying %s", dpidToStr(event.dpid))
def launch ():
  core.openflow.addListenerByName("ConnectionUp", _handle_ConnectionUp)
  log.info("Hub running.")
```
Table 1\. Hub Controller

### Useful POX API’s

*  ```connection.send( ... )``` function sends an OpenFlow message to a switch.

When a connection to a switch starts, a ConnectionUp event is fired. The above code invokes a `_handle_ConnectionUp ()` function that implements the hub logic.


*  ```ofp_action_output``` class

This is an action for use with `ofp_packet_out` and `ofp_flow_mod`. It specifies a switch port that you wish to send the packet out of. It can also take various "special" port numbers. An example of this, as shown in Table 1, would be `OFPP_FLOOD` which sends the packet out all ports except the one the packet originally arrived on.

Example. Create an output action that would send packets to all ports:

```bash
out_action = of.ofp_action_output(port = of.OFPP_FLOOD)
```


* ```ofp_match``` class (not used in the code above but might be useful in the assignment)

Objects of this class describe packet header fields and an input port to match on. All fields are optional -- items that are not specified are "wildcards" and will match on anything.

Some notable fields of ofp_match objects are:

1.  ```dl_src```- The data link layer (MAC) source address
2.  ```dl_dst```- The data link layer (MAC) destination address
3.  ```in_port```- The packet input switch port

Example. Create a match that matches packets arriving on port 3:

```python
match = of.ofp_match()  
match.in_port = 3
```

*  ```ofp_packet_out``` OpenFlow message (not used in the code above but might be useful in the assignment)

The ```ofp_packet_out``` message instructs a switch to send a packet. The packet might be one constructed at the controller, or it might be one that the switch received, buffered, and forwarded to the controller (and is now referenced by a `buffer_id`).

Notable fields are:

1.  ```buffer_id``` - The buffer_id of a buffer you wish to send. Do not set if you are sending a constructed packet.
2.  ```data``` - Raw bytes you wish the switch to send. Do not set if you are sending a buffered packet.
3.  ```actions``` - A list of actions to apply (for this tutorial, this is just a single ```ofp_action_output``` action).
4.  ```in_port``` - The port number this packet initially arrived on if you are sending by `buffer_id`, otherwise ```OFPP_NONE```.

Example. ```send_packet()``` method:

```python
def send_packet (self, buffer_id, raw_data, out_port, in_port):
  """
   Sends a packet out of the specified switch port.
   If buffer_id is a valid buffer on the switch, use that.  Otherwise,
   send the raw data in raw_data.
   The "in_port" is the port number that packet arrived on.  Use
   OFPP_NONE if you're generating this packet.
   """
  msg = of.ofp_packet_out()
  msg.in_port = in_port
  if buffer_id != -1 and buffer_id is not None:
    # We got a buffer ID from the switch; use that
    msg.buffer_id = buffer_id
  else:
    # No buffer ID from switch -- we got the raw data
    if raw_data is None:
      # No raw_data specified -- nothing to send!
      return
    msg.data = raw_data
  
  action = of.ofp_action_output(port = out_port)
  msg.actions.append(action)
  
  # Send message to switch
  self.connection.send(msg)
```

Table 2: Send Packet

1.  ```ofp_flow_mod``` OpenFlow message



This instructs a switch to install a flow table entry. Flow table entries match some fields of incoming packets, and executes some list of actions on matching packets. The actions are the same as for ```ofp_packet_out```, mentioned above (and, again, for the tutorial all you need is the simple ```ofp_action_output``` action). The match is described by an ```ofp_match``` object.



Notable fields are:

1.  idle_timeout - Number of idle seconds before the flow entry is removed. Defaults to no idle timeout.
2.  hard_timeout - Number of seconds before the flow entry is removed. Defaults to no timeout.
3.  actions - A list of actions to perform on matching packets (e.g., ofp_action_output)
4.  priority - When using non-exact (wildcarded) matches, this specifies the priority for overlapping matches. Higher values are higher priority. Not important for exact or non-overlapping entries.
5.  buffer_id - The buffer_id of a buffer to apply the actions to immediately. Leave unspecified for none.
6.  in_port - If using a buffer_id, this is the associated input port.
7.  match - An ofp_match object. By default, this matches everything, so you should probably set some of its fields!



Example. Create a flow_mod that sends packets from port 3 out of port 4.



```
fm = of.ofp_flow_mod()  
fm.match.in_port = 3  
fm.actions.append(of.ofp_action_output(port = 4))
```

## Verify Switch behavior with tcpdump



This time, let’s verify that hosts can ping each other when the controller is behaving like a Layer 2 learning switch. Kill the POX controller by pressing Ctrl-C in the window running the controller program and run the l2_learning example:



```bash
$ pox.py log.level --DEBUG forwarding.l2_learning
```

Like before, we'll create xterms for each host and view the traffic in each. In the Mininet console, start up three xterms:



```bash
mininet> xterm h1 h2 h3
```



Arrange each xterm so that they're all on the screen at once. This may require reducing the height of to fit a cramped laptop screen.



In the xterms for h2 and h3, run tcpdump, a utility to print the packets seen by a host:

```bash
# tcpdump -XX -n -i h2-eth0
```



and respectively:



```bash
# tcpdump -XX -n -i h3-eth0
```



In the xterm for h1, send a ping:



```bash
# ping -c 1 10.0.0.2
```



Here, the switch examines each packet and learn the source-port mapping. Thereafter, the source MAC address will be associated with the port. If the destination of the packet is already associated with some port, the packet will be sent to the given port, else it will be flooded on all ports of the switch.

You can close the xterms now.



# Assignment

## Background


A Firewall is a network security system that is used to control the flow of ingress and egress traffic usually between a more secure local-area network (LAN) and a less secure wide-area network (WAN). The system analyses data packets for parameters like L2/L3 headers (i.e., MAC and IP address) or performs deep packet inspection (DPI) for higher layer parameters (like application type and services etc) to filter network traffic. A firewall acts as a barricade between a trusted, secure internal network and another network (e.g. the Internet) which is supposed to be not very secure or trusted.



In this assignment, your task is to implement a layer 2 firewall that runs alongside the MAC learning module on the POX OpenFlow controller. The firewall application is provided with a list of MAC address pairs i.e., access control list (ACLs). When a connection establishes between the controller and the switch, the application installs flow rule entries in the OpenFlow table to disable all communication between each MAC pair.



## Network Topology

Your firewall should be agnostic of the underlying topology. It should take MAC pair list as input and install it on the switches in the network. To make things simple, we will implement a less intelligent approach and will install rules on all the switches in the network.  



## Handling Conflicts



POX allows running multiple applications concurrently i.e., MAC learning can be done in conjunction with firewall, but it doesn’t automatically handles rule conflicts. You have to make sure, yourself, that conflicting rules are not being installed by the two applications e.g., both applications trying to install a rule with same src/dst MAC at the same priority level but with different actions. The most simplistic way to avoid this contention is to assign priority level to each application.



## <a name="h.j57egno8inn0"></a><span class="c19">Understanding the Code



To start this assignment update the course's Github repo (by default, ```Coursera-SDN```) on your host machine using ```git pull```. Turn on your guest VM (if it is turned off) using ```vagrant up```. Now ssh into the guest VM using ```vagrant ssh```. Go to the directory with the updated code base in your guest VM. 
```bash
cd /vagrant/assignments/simple-controller
```
 It consists of three files:



1.  ```firewall.py```: a sekleton class which you will update with the logic for installing firewall rules.
2.  ```firewall-policies.csv```:  a list of MAC pairs (i.e., policies) read as input by the firewall application.
3.  ```submit.py```: used to submit your code and output to the coursera servers for grading.



You don’t have to do any modifications in ```firewall-policies.csv``` and ```submit.py```.



The firewall.py is populated with a skeleton code. It consists of a firewall class that has a _handle_ConnectionUp function. It also has a global variable, policyFile, that holds the path of the firewall-policies.csv file. Whenever a connection is established between the POX controller and the OpenFlow switch the _handle_ConnectionUp functions gets executed.



Your task is to read the policy file and update the _handle_ConnectionUp function. The function should install rules in the OpenFlow switch that drop packets whenever a matching src/dst MAC address (for any of the listed MAC pairs) enters the switch. <span class="c24">(Note: make sure that you handle the conflicts carefully. Follow the technique described in the section above)



## Testing your Code

Once you have your code, copy the firewall.py in the `$POX_LOCATION/pox/misc` directory on your VM. Also in the same directory create the following file:



```bash
$ cd $POX_LOCATION/pox/misc
$ touch firewall-policies.csv
```


and copy the following lines in it:


```csv
id,mac_0,mac_1
1,00:00:00:00:00:01,00:00:00:00:00:02
```


This will cause the firewall application to install a flow rule entry to disable all communication between host (h1) and host (h2).



Run POX controller:


```bash
$ cd $POX_LOCATION
$ pox.py forwarding.l2_learning misc.firewall 
```

This will run the controller with both MAC learning and firewall application.

Now run mininet:



```bash
$ sudo mn --topo single,3 --controller remote --mac
```


In mininet try to ping host (h2) from host (h1):


```bash
mininet> h1 ping -c 1 h2
```


What do you see? If everything has be done and setup correctly then host (h1) should not be able to ping host (h2).

Now try pinging host (h3) from host (h1):

```bash
mininet> h1 ping -c 1 h3
```

What do you see? Host (h1) is able to ping host (h3) as there is no flow rule entry installed in the network to disable the communication between them.

Finally, stop mininet and POX controller.

```bash
mininet> exit
```

```bash
$ sudo fuser -k 6633/tcp
```



## Submitting your Code

Upload (ONLY) the `firewall.py` and `firewall-policies.csv` files to the student portal.

## Acknowledgments

 This course has been adapted from Nick Feamster's [Coursera Course](https://github.com/noise-lab/Coursera-SDN/tree/master)
