+++
title = "Some notes on building and running mirage unikernels on Cubieboard2"
date = "2014-09-10"
slug = "2014/09/10/some-notes-on-building-and-running-mirage-unikernels-on-cubieboard2"
Categories = []
+++

*First a caveat: These notes reflect the state of Mirage as of the date of this post. Mirage is still in heavy development and the instructions here will change (hopefully become simpler) going forward.*

These are some notes on what I had to do to get a [Mirage](http://mirageos.org/) unikernel running under [Xen](http://www.xenproject.org/) on a [cubieboard2](http://cubieboard.org/). The intent here is to document some of the gotchyas I ran into.

**Installing Xen for ARM on Cubieboard**

First download the pre-built img file from: http://blobs.openmirage.org/cubieboard2.tar

Extract the img file:

    $ tar -xvf cubieboard2.tar

For the most part you can follow instructions [here](https://github.com/mirage/xen-arm-builder) to copy the cubieboard2.img to a micro SD card, but I'll add a little more detail here.

When you first plug in your empty micro SD card (I have a micro SD to USB adaptor) to your Linux computer you can discover which /dev it's on by issuing:

    $ fdisk -l

On my machine it showed up at /dev/sdb. Now use dd to copy the cubieboard2.img to the empty micrSD card:

    $ dd if=cubieboard2.img of=/dev/sdb

After the copy is complete, plug the micro SD card into your Cubuieboard2 and boot it up. It will boot into Linaro Linux (14.04) on DOM0. If you have a USB to TTL serial cable (I got one from AdaFruit here: http://www.adafruit.com/products/954 ) you can plug that into your cubieboard2 (white to TX, green to RX, black to GND - don't plug in the red wire to VCC).  Connect the other side of the cable to your Linux machine and then you can watch the cubieboard boot up by running:

    $ sudo screen /dev/ttyUSB0 115200

(NOTE: I'm running Fedora 20, not sure, but other distros may need a different device in place of /dev/ttyUSB0)

The cubieboard will get an address on your network using DHCP. Ater it finishes booting you can log in with username: mirage    password: mirage and then run ifconfig to see what address has been assigned to it.

If you don't have a USB to TTL serial cable you can check to see what address was assigned to the cubieboard by first (prior to powering up the cubieboard) issuing:

    $  sudo arp-scan -I p4p1 192.168.1.0/16

(substitute your subnet for 192.168.1.0 if that isn't the subnet of the network you're on)

Note the devices attached to each address. Now power up your cubieboard, wait a couple of minutes and then reissue the arp command to see what new address has been added - the new address will be the address assigned by DHCP to your cubieboard.

**Install mirage on the Cubieboard**

Now that your cubieboard has booted you need to install mirage. Fortunately, the cubieboard2.img already comes with OCaml 4.01 and opam installed.  Now install mirage:

    $ opam init
    $ opam install mirage-xen-minios
    $ opam pin mirage https://github.com/talex5/mirage.git#link_c_stubs
    $ opam pin mirage-xen https://github.com/mirage/mirage-platform
    $ opam pin tcpip https://github.com/talex5/mirage-tcpip.git#checksum
    $ opam install mirage    

**Try out a Mirage example**

I wanted to try out an example that uses networking on Xen which means that the OCaml TCPIP stack would be used.  There are several examples in the mirage-skeleton repo: https://github.com/mirage/mirage-skeleton

Now ssh into your cubieboard and clone mirage-skeleton:

    $ git clone git@github.com:philtomson/mirage-skeleton.git

(Notice that I'm pointing you to my mirage-skeleton repo, not the canonical one - I've made some changes to the stackv4 example config.ml so that you can easily specify DHCP which we'll see a bit later, in the future, those changes will hopefully be in the main mirage-skeleton repo)

Now we'll try out stackv4 which is a simple http server example.

    $ cd mirage-skeleton/stackv4

Do the mirage configure, but set DHCP so that Mirage will create a main.ml file that uses DHCP instead of getting the hardcoded 10.0.0.2 (hardcoded in mirage):

    $ DHCP=1 mirage configure --xen
    $ mirage build

Now you will have run into a problem (at least as of this date, hopefully this will be fixed in the near future). You'll see this error:

    stackv4      ld: cannot find -lpthread
    stackv4      ld: cannot find -lcrypto
    stackv4      ld: cannot find -lssl
    stackv4      make: build Error 1 

As it turns out, you don't need those libraries, so edit the Makefile to remove the lines :

    -lpthread \
    -lcrypto \
    -lssl \

Save the edited Makefile and run (again):

    $ mirage build
    $ make run

Notice the following message that comes from the *make run*:

    stackv4.xl has been created. _Edit it to add VIFs_ or VBDs
    Then do something similar to: xl create -c stackv4.xl
    
So now you need to edit the stackv4.xl, notice that it already has the following commented line:

    # vif = [ 'mac=c0:ff:ee:c0:ff:ee,bridge=br0' ]

Edit it to remove the comment (#) and get rid of the mac address (I do not know if it is correct and since it possibly isn't, and we don't need it, best to just get rid of it. Make it look like:

    vif = [ 'bridge=br0' ]

As an aside, where did the *br0* come from?  It's possible you need something else there. Certainly I've seen other documents that had another identifier there. In order to figure out what it should be you should run:

    $ brctl show

The result will be something like:

    bridge name	bridge id		STP enabled	interfaces
    br0		8000.02d908c0a687	no		eth0

The bridge name is what you're going to want in the vif line above. In my case the comment matched the output of *brctl* so no change needed. 

And now the moment we've been waiting for... start up that unikernel:

     sudo  xl create -c stackv4.xl   

If all went well, you should see that it reports the address it gets from DHCP:

     ...
     DHCP: offer received: 192.168.1.11
     ...
     DHCP: offer 192.168.1.11 255.255.255.0 [192.168.1.1]
      sg:true gso_tcpv4:true rx_copy:true rx_flip:false smart_poll:false
     ARP: sending gratuitous from 192.168.1.11
     DHCP offer received and bound
     Manager: configuration done
     IP address: 192.168.1.11

Now you can use curl on another machine on your network to see the hello world message:

    $ curl http://192.168.1.11
    hello mirage world!

And there you have it. 

**Shutting down the server**

To shut down that mirage unikernel server on the cubieboard, issue the following:

    $ sudo xl list
    Name                                        ID   Mem VCPUs	State	Time(s)
    Domain-0                                     0   512     2     r-----     194.3
    stackv4                                     11   256     1     -b----       0.1

*stackv4* is the one you want to stop (don't stop Domain-0 as that's the Linux instance running in DOM0 of Xen - the one you're typing commands into) 

    $ sudo xl destroy stackv4

  
        

    
    
    
    


