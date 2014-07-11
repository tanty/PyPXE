**NOTE:** This documentation is unfinished and is still a work in progress.

---

#Background
>The Preboot eXecution Environment (PXE, also known as Pre-Execution Environment; sometimes pronounced "pixie") is an environment to boot computers using a network interface independently of data storage devices (like hard disks) or installed operating systems. -[Wikipedia](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) 

PXE allows computers to boot from a binary image stored on a server, rather than the local hardware. Broadly speaking, a DHCP server informs a client of the TFTP server and filename from which to boot. 

##DHCP
In the standard DHCP mode, the server has been implemented from [RFC2131](http://www.ietf.org/rfc/rfc2131.txt), [RFC2132](http://www.ietf.org/rfc/rfc2132.txt), and the [DHCP Wikipedia Entry](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol).  

The DHCP server is in charge of assigning new clients with IP addresses, and informing them of the location of the TFTP server and filename they should look for. The top half of the DHCP request, as seen on the Wikipedia entry, consists of some network information, followed by 192 legacy null bytes, and then the magic cookie.  

After the magic cookie, there are optional fields. These fields are defined by the field ID, a length byte and then the option data. Every DHCP packet has an option 53 field. This field specifies the packet type, be it DISCOVERY, OFFER, REQUEST or ACKNOWLEDGEMENT.  

The DISCOVERY and REQUEST packets are those sent by the client to the broadcast address (IPv4: 255.255.255.255) where the client sends this data on port 68 and the server receives it on port 68. The OFFER and ACKNOWLEDGEMENT packets are sent from the server to the broadcast address where the server sends this data on port 67 and the client receives it on port 68. These four packets make up the four-way handshake.  

The OFFER and ACKNOWLEDGEMENT packets both contain the same option data for each client. This can include the router, subnet mask, lease time, and DHCP server IP address.

Also included in these options are our PXE options. The minimum required option fields are option 66 and option 67. Option 66 denotes the IP of the TFTP server, and option 67 denotes the filename of the file to retrieve and boot.  

Once the four way handshake is complete, the client will send a TFTP read request to the given fileserver IP address requesting the given filename.

###ProxyDHCP
ProxyDHCP mode is useful for when you either cannot (or do not want to) change the main DHCP server on a network. The bulk of ProxyDHCP information can be found in the [Intel PXE spec](http://www.pix.net/software/pxeboot/archive/pxespec.pdf). The main idea behind ProxyDHCP is that the main network DHCP server can hand out the IP leases while the ProxyDHCP server hands out the PXE information to each client. Therefore, slightly different information is sent in the ProxyDHCP packets.

There are multiple ways to implement ProxyDHCP: broadcast, multicast, unicast or lookback. Lookback is the simplest implementation and this is what we have chosen to use. When we receive a DHCP DISCOVER from a client, we respond with a DHCP OFFER but the OFFER packet is sent without a few fields we would normally send in standard DHCP mode (this includes an offered IP address, along with any other network information such as router, DNS server(s), etc.). What we include in this OFFER packet (which isn't in a normal DHCP packet), is a vendor-class identifier of 'PXEClient' - this string identifies the packet as being relevant to PXE booting.

There are a few vendor-specific options under the DHCP option 43:
* The first of these options is PXE discovery control; this is a bitmask defined in the PXE spec. When bit 3 is set, the PXE client will look backwards in the packet for the filename. The filename is located in the standard 'DHCP Boot Filename' area, previously referred to as 'legacy'. This is a NULL-terminated string located at offset 150 in the DHCP packet (before the DHCP magic cookie).
* The second vendor-specific option that is used is the PXE menu prompt. Although not visible, this is required by the spec and causes problems if it is missing. The main use for this is for the other DISCOVERY modes.  

The client should receive two DHCP OFFER packets in ProxyDHCP mode: the first from the main DHCP server and the second from the ProxyDHCP server. Once both are received, the client will continue on with the DHCP handshake and, after it is complete, the client will boot using the settings in the DHCP OFFER from the ProxyDHCP server.

##TFTP
We have only implemented the read OPCODE for the TFTP server, as PXE does not use write. The main TFTP protocol is defined in [RFC1350](http://www.ietf.org/rfc/rfc1350.txt)

###blksize
The blksize option, as defined in [RFC2348](http://www.ietf.org/rfc/rfc2348.txt) allows the client to specify the block size for each transfer packet. The blksize option is passed along with the read opcode, following the filename and mode. The format is blksize, followed by a null byte, followed by the ASCII base-10 representation of the blksize (i.e 512 rather than 0x200), followed by another null byte.

##HTTP
We have implemented GET and HEAD, as there is no requirement for any other methods. The referenced RFCs are [RFC2616](http://www.ietf.org/rfc/rfc2616.txt) and [RFC7230](http://www.ietf.org/rfc/rfc7230.txt).  

The HEAD method is used by some PXE ROMs to find the Content-Length before the GET is sent.

#PyPXE Services
Each different service implemented (TFTP, DHCP, and HTTP) resides in its own file in the root of the repository. You can call/configure them independently if you're like to use PyPXE as a library. See ```server.py``` in the root of the repo for example usage on how to call, define, and setup the services. When running any Python script that uses these classes, it should be run as a user with root privileges as they bind to interfaces and without root privileges the services will most likely fail to bind properly.

##TFTP Server (tftpd.py)
The TFTP server class, ```TFTPD()``` requires three optional parameters be set in order to be constructed:
* ```ip``` - this is the IP address that the TFTP server will bind to, by default it is set to '0.0.0.0' so that it binds to all available interfaces.
* ```port``` - this it the port that the TFTP server will run on, by default the port is 69 as that is the default port for TFTP.
* ```netbootDirectory``` - this is the directory that the TFTP server will serve files from similarly to that of ```/tftpboot```, by default it is set to '.' (current directory)

##Additional Information
The function ```chr(0)``` is used in multiple places throughout the servers. This denotes a ```NULL``` byte, or ```\x00```