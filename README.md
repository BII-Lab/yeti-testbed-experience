# Experiences from Root Testbed in the Yeti DNS Project 


<Note it is Davey's edit after the review of ISE. For latest version 
going to be submited please refer to the xml and txt version in this 
repo.>

### **Abstract**

   DNS root system is the apex node of the DNS Tree-structure. Due 
   to its importance and special roll, it is hard to test and 
   implement new ideas. This document discusses our experiences 
   from the operation and experiments in a live IPv6-only root testbed, 
   in the Yeti DNS Project. The purpose of Yeti testb is to provide 
   a place to try new or stupid ideas on DNS root system. The 
   document covers the experiments findings in IPv6-only operation, 
   KSK rollover and multiple signers expereiments. It also records 
   and discusses the practical experiences, obstacle and opportunities 
   for this type of root system setup. 
   
   <!--The document also makes some recommendations on -->
   <!--future root system design and future work that is needed take into -->
   <!--consideration regarding the issues on IPv6 fragmentaion.-->

   REMOVE BEFORE PUBLICATION: Although this document is submitted as an
   independent submission, comments are welcome in the IETF DNSOP (DNS
   Operations) working group mailing list.  The source of the document
   is currently placed at GitHub [xml-file].

## 1. Introduction


   [RFC1034] says the domain name space is a tree structure. The top
   level of the tree (root zone) for the unique identifier system is 
   served by the DNS root system. It is special in some aspects. The 
   most notable one is that DNS root system is the entrance of the 
   DNS system for most of resolvers. The root zone is authoritative 
   for all gTLD and ccTLD. It is pivotal in both technical and political 
   aspects making the current Internet useful and valuable. In the history 
   of DNS root server, great efforts had been taken to 
   implement new technologies, such as root name compression, Anycast 
   instance, IPv6 and DNSSEC support in root zone and operation. However, 
   it become more and more difficult to test and implement new ideas 
   on current operational system due to the increasing scale and 
   influence of Internet nowadays. 
   
   In order to provide a place to try new idea even stupid ones, a live 
   DNS root system call Yeti was built with 25 divers servers distributed 
   globally which is different from Lab environment. Some innovative ideas 
   were tried like IPv6-only root, DNSSEC key roll and management, shared 
   zone control [] etc. This document is a informative memo about the Yeti 
   testbed and the experiments (more than one) done on the testbed. It 
   covers the experiments findings in IPv6 fragmentation, KSK rollover and 
   multiple signers expereiments as well as some DNS software bugs during 
   the test. The memo also record some discussion and recommendations on 
   future root operation.
   
   The targeted audiences of this memo can be the experienced people who 
   are intereted in Yeti Testbed and experiments. The audiences without 
   speciall background on DNS root can follow our introduction to better 
   understand the root system and captures the issues in this field.
   
   Note that this memo documents our experiences primarily from May 2015 to 
   December of 2016. During the time frame, IANA transition process was done, 
   ICANN launched its KSK rollover plan and VeriSign increased it's ZSK length 
   from 1024 bits to 2048 bits. So some text are records of expereinces at the 
   time of writing.
   
   The rest of this document is organized as follows. Section 2 introduces 
   the terminology and how root system works. Section 3 discribes the system 
   and network setup. Sections 4 discusses the expereince and findings in some 
   experiments. Section 5 presents some of our ideas for future work. Section 6 
   draws conclusions and makes recommendations on future root system design.
   

## 2. Technology and Terminology
### 2.1 Technology and Terminology

   In this document, the following terms are used. 
   
   * Domain Name System Security Extensions (DNSSEC)： DNSSEC provides 
   authentication of DNS data and keep data integrity by using public-key 
   cryptography, as defined by [RFC4033], [RFC4034], [RFC4035].

   * Zone signing key(ZSK): "DNSSEC keys that can be used to sign all the 
   RRsets in a zone that require signatures, other than the apex DNSKEY 
   RRset."  ([RFC6781], Section 3.1) By defination the ZSK of root zone is 
   the keys that sign all RRsets in root zone, other than the apex DNSKEY 
   RRset.
   
   * Key signing key (KSK): DNSSEC keys that "only sign the apex DNSKEY RRset 
   in a zone." (Quoted from [RFC6781], Section 3.1) By definiation the KSK 
   of root zone is the keys that only sign apex DNSKEY RRset in the root zone.
   
   * Priming Queries: "Priming is the act of finding the list of root servers from a
   configuration that lists some or all of the purported IP addresses of
   some or all of those root servers.  A recursive resolver starts with
   no information about the root servers, and ends up with a list of
   their names and their addresses." [Quoted from rfc8109, section 2]
   
   * Hint file: a configuration file that lists some or all the purported IP 
   addresses of some or all of those root servers. DNS implemented like BIND 
   and Unbound use hint file to launch the priming queries, but knot uses a 
   specific format for the hints.

   It is strongly recommaned for audiences to read RSSAC Lexico document. 
   It introduces the most relevent terminologies such as : root zone, 
   root service, root server, root instance etc. In this memo, the terms 
   of root operational roles are introduced belows:
   
   * Root server operator: A root server operator is an organization 
   responsible for managing the root service on IP addresses specified 
   in the root zone and the root hints file. Currently there are 12 root 
   server operators running 13 letters from A to M (root-servers.org)

   * Root zone administrator. The root zone administrator manages the data 
   contained within the root zone, which involves assigning the operators 
   of top-level domains, such as .uk and .com, and maintaining their technical 
   and administrative details. Right now, ICANN's affiliate PTI who runs 
   IANA function is the root zone administrator.

   * Root zone maintainer: The root zone maintainer is responsible for 
   accepting service data from the root zone administrator, formatting 
   it into zone file format, cryptographically signing it using the Zone 
   Signing Key (ZSK) for the root zone, and putting it into the root zone 
   distribution system. Before and after IANA transition, the US. company 
   VeriSign serves as the root zone maintainer with a contract with ICANN. 
   
   * Root zone distribution master/system: The root zone distribution 
   master(DM) is the DNS Primary server systems (or a colletion of 
   systems and procedures) by which root servers acquire root zone in 
   a secure transmission. 
   
   Note that in the RSSAC Lexico document, the terms like master and 
   hidden master are deprected and root zone distribution system is 
   used instead. But at the time of Yeti testbed was buit, VeriSign take 
   the roles of DM as well as root zone maintainer. So in this memo, 
   we still use DM to highlight the way how root zone is published and 
   its importance.  
   
<!-- ## 3. problem statement -->
<!--   When Yeti testbed idea was firstly conceived, there are several -->
<!--   challenges In DNSSEC, IPv6 , root scalability and distribution. -->

<!--   * DNSSEC operation: IANA rolls the ZSK every six weeks but the KSK has -->
<!--	  never been rolled at that time. Is RFC5011 [RFC5011] widely supported -->
<!--      by resolvers? How about larger key size or different encryption algorithm?  -->
<!--      Is the DNS packet size limitation (512 or 1280 bytes) should be respected -->
<!--      during KSK rollover nowadays?-->

<!--   * IPv6-only capability: IPv6 fragmentation is different from  IPv4 [Fragmenting-IPv6].  -->
<!--      It is not clear whether DNS can survive  without IPv4 (in an IPv6-only -->
<!--      environment), or what impact in IPv6-only environment introduces to current -->
<!--      DNS operations. For instance, the IPv6 fragmentation may become an issue -->
<!--      during KSK rollover which introduces large response packet. -->

<!--    * Root scalability and distribution: There are 13 root server (12 root operator) with -->
<!--       600+ anycast instances in the globe as the time of writing. It is not easy to make -->
<!--       assertion that the number of root servers is enough or not. But in the aspects of -->
<!--       distribution of root sever operators and instances, there are some concerns on -->
<!--       external dependency and surveillance risk [Yeti-problem-statement].  -->

<!--     * there are arguments often accompanied with Root server topic in -->
<!--  political sense that some countries may think that they are left behind , as they were -->
<!--  not engaged in the key Internet infrastructure operation purely due to the history -->
<!--  reason. In consequence, fears of uncertainty and insecurity arises. Unfortunate, this -->
<!--  appeal is usually ignored and confront with less interests in technical community when -->
<!--  it gets closer to Internet governance field. -->

<!--   The author of this document hold the idea the a good design should consider both -->
<!--   technical and political issues. We document political issue is just for the audience to -->
<!--   understand the design of Yeti testbed and its experiment.-->

<!--**Note that**  ICANN KSK Plan， Bigger ZSK is implemented.-->

## 3. Yeti Testbed Setup


   To use the Yeti testbed operationally, the information that is
   required for correct root name service is a matching set of the
   following:

   o  a root "hints file"

   o  the root zone apex NS record set
   
   o  the root zone's signing key
   
   o  root zone trust anchor
	
   Although Yeti root testbed publishes strictly the same IANA 
   information for TLD data and meta-data, it is necessary to use 
   a special hint file to list the domain names and IP address 
   of Yeti root servers, which will enable the resolves initialize 
   themself by emitting priming queries. In addition to make 
   DNSSEC workable in Yeti root testbed, a set of signing key 
   (both KSK and ZSK) for Yeti root zone is introduced. 
   
   Below is a figure to demonstrate the topology of Yeti and the basic
   data flow, which consists of the Yeti distribution master, Yeti root
   server, and Yeti resolver:

	                           +------------------------+
	                         +-+     IANA Root Zone     +--+
	                         | +-----------+------------+  |
	   +-----------+         |             |               | IANA root zone
	   |    Yeti   |         |             |               |
	   |  Traffic  |      +--v---+     +---v--+      +-----v+
	   | Collection|      |  BII |     | WIDE |      | TISF |
	   |           |      |  DM  |     |  DM  |      |  DM  |
	   +---+----+--+      +------+     +-+----+      +---+--+
	       ^    ^         |              |               |
	       |    |         |              |               |   Yeti root zone
	       |    |         v              v               v
	       |    |   +------+      +------+               +------+
	       |    +---+ Yeti |      | Yeti |  . . . . . .  | Yeti |
	       |        | Root |      | Root |               | Root |
	       |        +---+--+      +---+--+               +--+---+
	       |            |             |                      |
	       | pcap       ^             ^                      ^ DNS lookup
	       | upload     |             |                      |
	       |
	       |                   +--------------------------+
	       +-------------------+      Yeti Resolvers      |
	                           |     (with Yeti Hint)     |
	                           +--------------------------+
	
 					  Figure 1.  The topology of Yeti testbed



### 3.1.  Distribution Master and Yeti root zone

   Yeti Distribution Master (DM) is a critical roll in Yeti root testbed for 
   Yeti root zone generation and distributing it to Yeti root server. 
   As shown in figure 1, the Yeti DM firstly takes the IANA root zone from 
   the source of root zone. The appendix-A of RFC7706 introduces the method 
   as well as root zone source from ICANN and 5 other root servers (B, C, F, G, K).
   
   Yeti DM performs minimal changes needed to serve the zone from the Yeti
   root servers instead of the IANA root servers.  In addition, different 
   from the IANA root, Yeti root server only serve the Yeti root zone. 
   No root-servers.org zone and .arpa zone are served. For more inforation 
   of Yeti root zone, the audiences can visit a monitoring page [Yeti-Mon] 
   showing the "Yeti root zone diff with IANA". 
   
   Yeti DM is also reponsible to implement DNSSEC on Yeti root zone by signing 
   the zone with the dnskey set of Yeti which is different from IANA dnskey set. 
   This signed root zone is then ready for zone transfer queries from Yeti 
   root servers.

   So the zone generation process is in the following sequence:

	   o  DM downloads the latest IANA root zone at a certain time
	   o  DM makes modifications to change from the IANA root servers 
	      to Yeti root servers
	   o  DM signs the new Yeti root zone with Yeti key
	   o  DM publishes the new Yeti root zone to Yeti root servers

   While in principle this could be done by a single DM, Yeti uses a set
   of three DMs to avoid single point of failure as well as the policical 
   consideration that avoid any sense that Yeti root testbed is run by a
   single organization. Each of three DMs independently fetches the
   root zone from IANA, signs it and publishes the latest zone data to
   Yeti root servers.

   In the same while, these DMs coordinate their work so that the Yeti root 
   zone is always consistent. The coordination of three DMs are one of 
   important expereiments in Yeti testbed which is introduced in section 4.1.
   
   For the security reason, DNSSEC requires key maintainer to roll 
   periodically. In the setting of Yeti, each DM is able to manage their ZSK 
   independently on the length of key, encryption algorithm and Key roll 
   timings etc.) But each DM is currently dependent on a single trust anchor. 
   The Yeti KSK rolling is introduced in section 4.3.

### 3.2.  Yeti Root Servers

   In the beginning of Yeti root testbed setup, we call for donators and 
   operaters of Yeti root server. There are only two basic requirement: 1) a 
   server with good IPv6 Internet access because Yeti root testbed is working
   purely in IPv6, 2) a dedicated domain name of the root server which is 
   configured as a slave to the Yeti distribution masters (DM). Appendix B 
   provides an introduction to setup a root server in Yeti root testbed 
   using popular DNS impelmentations like BIND, Knot and NSD. 
   
   Regarding the name scheme for individual server, there are two options. 
   One is to use separate and normal domains for root servers as shown 
   in Appendix A. It is current name scheme deployed in Yeti testbed. The 
   other one is to use a special non-delegated TLD, like bii.yeti-dns for 
   root server operated by BII. We found that modification of IANA root 
   zone by adding a new TLD is so controversial even for scientific purpose. 
   So we abandon the experience on latter one. Section 4.5 introduces more 
   discussion on the experiement on Root naming schemes.
  
   Since Yeti is a scientific research project, it needs to capture DNS
   traffic sent to one of the Yeti root servers for later analysis.
   Today some servers use dnscap, which is a DNS-specific tool to
   produce pcap files. There are several versions of dnscap floating
   around; some people use the VeriSign one.  Since dnscap loses packets
   in some cases (tested on a Linux kernel), some people use pcapdump.
   It requires the patch attached to this bug report [pcapdump-bug-report]

   By a survey we observed a good diversity on the Yeti root testbed. Among 
   the Yeti root servers, both VPS and pysicial machines are used. Most 
   operators work on Linux including Ubuntu, Debian, CentOS, ArchLinux, a few 
   use FreeBSD and NetBSD. One operator try Windows server and Windows DNS for 
   the purpose of test. Regardsing DNS software, most of them use BIND 
   (varying from 9.9.7 to 9.10.3). NSD (4.10 and 4.15), Knot (2.0.1 and 
   2.1.0) and even Bundy (1.2.0) are also the choice at the time of 
   this survey.
   
   Note that at the time of writing, there are 25 Yeti root servers distributed 
   around the world. These servers are selected taken their geo-location 
   into consideration. 
   
### 3.3.  Yeti Resolvers and Experimental Traffic

   In client side of Yeti DNS Project, there are DNS resolvers with IPv6
   support, updated with Yeti "hints" file to use the Yeti root servers
   instead of the IANA root servers, and using Yeti KSK as trust anchor.
   The Yeti KSK rollover is expected to change key during the experiment, 
   so it is required that resolver operator to configure the resolver 
   compliant to RFC 5011 for automatic update. It is also encouraged that 
   Yeti reslover impelments RFC8145, a mechanism that end-system resolvers 
   to signal to a server about their DNSSEC key status.

   Participants and volunteers are expected from individual researchers,
   labs of universities, companies and institutes, and vendors (for
   example, the DNS software implementers), developers of CPE devices &
   IoT devices, and middle box developers who can test their products
   and connect their own testbed into Yeti testbed.  Resolvers donated
   by Yeti volunteers are required to be configured with Yeti hint file
   and Yeti DNSSEC KSK.  It is required that Yeti resolver can speak
   both IPv4 and IPv6, given that not all authoritative servers on the
   Internet are IPv6 capable.

   At the time of writing several universities and labs have joined us
   and contributed certain amount of traffic to Yeti testbed.  To
   introduce desired volume of experiment traffic, Yeti root testbed adopted
   two alternative ways to increase the experimental traffic in the Yeti
   testbed and check the functionality of Yeti root system.

   One approach is to mirror the real DNS query to IANA root system by
   off-path method and replay it into Yeti testbed; this is implemented
   by some Yeti root server operators.  Another approach is to use
   traffic generating tool such as RIPE Atlas probes to generate
   specific queries against Yeti servers.
   
## 4. Experience and findings on Yeti Experiments

   As introduced, Yeti root testbed is designed for testing new ideas. 
   Some operational expereiments which are different from the current 
   practice of IANA root server system are impelmented. For instance 
   differnt naming scheme is used for individual root servers. Three-DM model 
   instead of single-DM model is tested. We rolled ZSK as well as KSK 
   in the testbed. We also encountered IPv6 fragmentaion issues in IPv6-only 
   environment.  

### 4.1 Root Naming S
