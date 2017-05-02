# Experiences from Root Testbed in the Yeti DNS Project

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

### 4.1 Root Naming Scheme
   In root server history, the naming scheme for individual root servers
   was not fixed. Current IANA Root server adopt [a-m].root-servers.net
   naming scheme to represent 13 servers which are labeled with letter
   from A to M. The authoritativeness is achieved by hosting "root-
   servers.net" zone in every root server. One reason behind this 
   naming scheme is that DNS label compression can be used to produce a
   smaller DNS response within 512 bytes. But there are still two conerns 
   on this naming schemes:
   
   o  Currently root-servers.net is not signed.  Kaminsky-like attacks
      are still possible for the the important information of Root
      server.
      
   o  The dependency to a single name(i.e..net) make the root system
      fragile in extreme case that all .net servers are down or
      unreachable but the root server still alive.

   In Yeti testbed there is a chance to design and test alternative naming 
   schemes to solve some issues with current naming scheme. 
   
   
#### 4.1.1 Expereince on Yeti Root Naming scheme 
   
   At the time of writing, Yeti is using separate and normal domains 
   for root servers (shown in Appendix A). It gets rid of the dependency 
   on the single name and intentionally produces larger packets for 
   priming responses for less name compression efficiency. Note that 
   the Yeti root testbed generate a repsonse up to 1757 Bytes to priming 
   query. 
   
   It is a issue for this naming scheme in which the priming
   response may not contain all glue record for Yeti Root servers.  It
   is documented as a technical findings [Yeti-glue-issue].  There are
   two approaches to solve the issue: one is to patch BIND 9 to includes
   the glue records in the additional section.  The other one is to add
   a zone file for each root server and answer for all of them at each
   Yeti server.  That means each Yeti root server would have a small
   zone file for "bii.dns-lab.net", "yeti-ns.wide.ad.jp", "yeti-
   ns.tisf.net", and so on.

#### 4.1.2 Discussion on .yeti-dns 
   Another naming scheme under Yeti lab test is to use a special non-
   delegated TLD, like .yeti-dns for root server operated by BII.  The
   benefit of non-delegated TLD naming scheme are in two aspects: 1) the
   response to a priming query is protected by DNSSEC; 2) To meet a
   political reason that the zone authoritative for root server is not
   delegated and belong to particular companies or organizations except
   IANA; 3) reduce the dependency of root server names to other DNS
   service; 4) to mitigate some kind of cache poisoning activities.

   The obvious concern of this naming scheme is the size of the signed
   response with RRSIG for each root server and optionally DNSKEY RRs.
   There is a Lab test result regarding the different size of priming
   response in Octet : 1) with no additional data, with RRISG in
   additional section , with DNSKEY+RRSIG in additional section (7 keys
   in MZSK experiment.  MZSK is to be described in section 4.2)

             +--------------------+--------+----------------+
             | No additional data | RRSIG  | RRISG +DNSKEY  |
             +--------------------+--------+----------------+
             | 753                | 3296   | 4004           |
             +--------------------+--------+----------------+

   We found that modification of IANA root zone by adding a new TLD is
   so controversial even for scientific purpose.  There are non-trivial
   discussions on this issue in Yeti discuss mailing list, regarding the
   proposal .yeti-dns for root name or .local for new AS112
   [I-D.bortzmeyer-dname-root].  It is argued that this kind of
   experiment should based on community consensus from technical bodies
   like IETF and be operated within a limited duration in some cases.

   Note that a document named "Technical Analysis of the Naming Scheme
   used for Individual Root Servers" is being developed in RSSAC Caucus.
   And it will be published soon


### 4.2 Multiple DMs coordination

#### 4.2.1.  Yeti root zone SOA SERIAL

   Consistency with IANA root zone except the apex record is one of most
   important point for the project.  As part of Yeti DM design, the Yeti
   SOA SERIAL which reflects the changes of yeti root zone is one factor
   to be considered.

   Currently IANA SOA SERIAL number for root zone is in the form of
   YYYYMMDDNN, like 2015111801.  In Yeti root system, IANA SOA SERIAL is
   directly copied in to Yeti SOA SERIAL.  So once the IANA root zone
   has changed with a new SOA SERIAL, a new version of the Yeti root
   zone is generated with the same SOA SERIAL.

   There is a case of Yeti DM operation that when a new Yeti root server
   added, DM operators change the Yeti root zone without change the SOA
   SERIAL which introduces inconsistency of Yeti root system.  To avoid
   inconsistency, the DMs publish changes only when new IANA SOA SERIAL
   is observed.

   An analysis of IANA convention shows IANA SOA SERIAL change twice a
   day (NN=00, 01).  Since October 2007 the maximum of NN was 03 while
   NN=2 was observed in 13 times.

#### 4.2.2.  Timing of Root Zone Fetch

   Yeti root system operators do not receive notify messages when IANA
   root zone is updated.  So each Yeti DM checks the root zone serial
   periodically.  At the time of writing, each Yeti DM checks to see if
   the IANA root zone has changed hourly, on the following schedule:

                         +-------------+---------+
                         | DM Operator | Time    |
                         +-------------+---------+
                         | BII         | hour+00 |
                         | WIDE        | hour+20 |
                         | TISF        | hour+40 |
                         +-------------+---------+

   Note that Yeti DMs can check IANA root zone more frequently (every
   minute for example).  A test done by Yeti participant shows that the
   delay of IANA root zone update from the first IANA root server to
   last one is around 20 minute.  Once a Yeti DM fetch the new root
   zone, it will notify all the Yeti root servers with a new SOA serial
   number.  So normally Yeti root server will be notified in less than
   20 minute after new IANA root zone generated.  Ideally, if an IANA DM
   notifies the Yeti DMs, Yeti root zone will be updated in more timely
   manner.

#### 4.2.3.  Information Synchronization

   Given three DMs operational in Yeti root system, it is necessary to
   prevent any inconsistency caused by human mistakes in operation.  The
   straight method is to share the same parameters to produce the Yeti
   root zone.  There parameters includes following set of files:

   * the list of Yeti root servers, including:
      * public IPv6 address and host name
      * IPv6 addresses originating zone transfer
   * IPv6 addresses to send DNS notify to
	* The ZSKs used to sign the root
	* The KSK used to sign the root
	* the SERIAL when this information is active

   The operation is simple that each DM operator synchronize the files
   with the information needed to produce the Yeti root zone.  When a
   change is desired (such as adding a new server or rolling the ZSK), a
   DM operator updates the local file and push to other DM.  A SOA
   SERIAL in the future is chosen for when the changes become active.
   
#### 4.2.4 Experience
 
  　　ＴＢＤ

### 4.3 Multiple-Signers

   According to the Problem statement of Yeti DNS Project, more
   independent participants and operators of the root system is
   desirable.  As the name implies, multi-ZSK (MZSK) mode introduces
   different ZSKs sharing a single unique KSK, as opposed to the IANA
   root system (which uses a single ZSK to sign the root zone).  On the
   condition of good availability and consistency on the root system,
   the Multi-ZSK proposal is designed to give each DM operator enough
   room to manage their own ZSK, by choosing different ZSK, length,
   duration, and so on; even the encryption algorithm may vary (although
   this may cause some problem with older versions of the Unbound
   resolver).

#### 4.3.1.  MZSK lab experiment

   In the lab test phase, we simply setup two root servers (A and B) and
   a resolver switch between them (BIND only).  Root A and Root B use
   their own ZSK to sign the zone.  It is proved that Multi-ZSK works by
   adding multiple ZSK to the root zone.  As a result, the resolver will
   cache the key sets instead of a single ZSK to validate the data no
   matter it is signed by Root A or Root B.  We also tested Unbound and
   the test concluded in success with more than 10 DMs and 10 ZSKs.

   Although more DMs and ZSKs can be added into the test, adding more
   ZSKs to the root zone enlarges the DNS response size for DNSKEY
   queries which may be a concern given the limitation of DNS packet
   size.  Current IANA root server operators are inclined to keep the
   packets size as small as possible.  So the number of DM and ZSK will
   be parameter which is decided based on operation experience.  In the
   current Yeti root testbed, there are 3 DMs, each with a separate ZSK.

### 4.3.2.  MZSK Yeti experiment

   After the lab test, the MZSK experiment is being conducted on the
   Yeti platform.  There are two phases:

   o  Phase 1.  In the first phase, we confirmed that using multiple
      ZSKs works in the wild.  We insured that using the maximum number
      of ZSKs continues to work in the resolver side.  Here one of the
      DM (BII) created and added 5 ZSKs using the existing
      synchronization mechanism.  (If all 3 ZSKs are rolling then we
      have 6 total.  To get this number we add 5.)

   o  Phase 2.  In the second phase, we delegated the management of the
      ZSKs so that each DM creates and publishes a separate ZSK.  For
      this phase, modified zone generation protocol and software was
      used [Yeti-DM-Sync-MZSK], which allows the DM to sign without
      access to the private parts of ZSKs generated by other DMs.  In
      this phase we roll all three ZSKs separately.

   The MZSK experiment was finished by the end of 2016-04.  Almost
   everything appears to be working.  But there have been some findings
   [Experiment-MZSK-notes], including discovering that IPv6 fragmented
   packets are not forwarded on an Ethernet bridge with netfilter
   ip6_tables loaded on one authority server, and issue with IXFR
   falling back to AXFR due to multiple signers which will be detailed
   in a separate draft describing as a problem statement.


### 4.4 KSK Rollover 

   The Yeti DNS Project provides a good basis to conduct a real-world
   experiment of a KSK rollover in the root zone.  It is not a perfect
   analogy to the IANA root because all of the resolvers to the Yeti
   experiment are "opt-in", and are presumably run by administrators who
   are interested in the DNS and knowledgeable about it.  Still, it can
   inform the IANA root KSK roll.

   The IANA root KSK has not been rolled as of the writing.  ICANN put
   together a design team to analyze the problem and make
   recommendations.  The design team put together a plan[ICANN-ROOT-ROLL].  
   The Yeti DNS Project may evaluate this
   scenario for an experimental KSK roll.  The experiment may not be
   identical, since the time-lines laid out in the current IANA plan are
   very long, and the Yeti DNS Project would like to conduct the
   experiment in a shorter time, which may considered much difficult.

   The Yeti KSK is rolled twice in Yeti testbed as of the writing.  In
   the first trial, it made old KSK inactive and new key active in one
   week after new key created, and deleted the old key in another week,
   which was totally unaware the timer specified in RFC5011.  Because
   the hold-down timer was not correctly set in the server side, some
   clients (like Unbound) receive SERVFAILs (like dig without +cd)
   because the new key was still in AddPend state when old key was
   inactive.  The lesson from the first KSK trial is that both server
   and client should compliant to RFC5011.

   For the second KSK rollover, it waited 30 days after a new KSK is
   published in the root zone.  Different from ICANN rollover plan, it
   revokes the old key once the new key become active.  We don't want to
   wait too long, so we shorten the time for key publish and delete in
   server side.  As of the writing, only one bug [KROLL-ISSUE]spotted on
   one Yeti resolver (using BIND 9.10.4-p2) during the second Yeti KSK
   rollover.  The resolver is configured with multiple views before the
   KSK rollover.  DNSSEC failures are reported once we added new view
   for new users after rolling the key.  By checking the manual of
   BIND9.10.4-P2, it is said that unlike trusted-keys, managed-keys may
   only be set at the top level of named.conf, not within a view.  It
   gives an assumption that for each view, managed-key can not be set
   per view in BIND.  But right after setting the managed-keys of new
   views, the DNSSEC validation works for this view.  As a conclusion
   for this issue, we suggest currently BIND multiple-view operation
   needs extra guidance for RFC5011.  The manage-keys should be set
   carefully during the KSK rollover for each view when the it is
   created.

   Another of the questions of KSK rollover is how can an authority
   server know the resolver is ready for RFC5011.  Two Internet-Drafts
   [I-D.wessels-edns-key-tag] and [I-D.wkumari-dnsop-trust-management]
   try to address the problem.  In addition a compliant resolver
   implementation may fail without any complain if it is not correctly
   configured.  In the case of Unbound 1.5.8, the key is only readable
   for DNS users [auto-trust-anchor-file].

### 4.5 IPv6 fragments

   There are two cases in Yeti testbed reported that some Yeti root
   servers failed to pull the zone from a Distribution Master via AXFR/
   IXFR.  Two facts have been revealed in both client side and server
   side after trouble shooting.

   One fact in client side is that some operation system can not handle
   IPv6 fragments correctly and AXRF/IXFR in TCP fails.  The bug covers
   several OSs and one VM platform (listed below).

                +-----------------------+-----------------+
                | OS                    | VM              |
                +-----------------------+-----------------+
                | NetBSD 6.1 and 7.0RC1 | VMware ESXI 5.5 |
                | FreeBSD10.0           |                 |
                | Debian 3.2            |                 |
                +-----------------------+-----------------+

   Another fact is from server side in which one TCP segment of AXRF/
   IXFR is fragmented in IP layer resulting in two fragmented packets.
   This weird behavior has been documented IETF
   draft[I-D.andrews-tcp-and-ipv6-use-minmtu].  It reports a situation
   that some implementations of TCP running over IPv6 neglect to check
   the IPV6_USE_MIN_MTU value when performing MSS negotiation and when
   constructing a TCP segment.  It will cause TCP MSS option set to 1440
   bytes, but IP layer will limit the packet less than 1280 bytes and
   fragment the packet to two fragmented packets.

#### 4.5.1 DNS Fragments

   In consideration of new DNS protocol and operation, there is always a
   hard limit on the DNS packet size.  Take Yeti for example: adding
   more root servers, using the Yeti naming scheme, rolling the KSK, and
   Multi-ZSK all increase the packet size.  The fear of large DNS
   packets mainly stem from two aspects: one is IP-fragments and the
   other is frequently falling back to TCP.

   Fragmentation may cause serious issues; if one of the fragment is
   lost at random, it results in the loss of entire packet and involve
   timeout.  If the fragment is dropped by a middle-box, the query
   always results in failure, and result in name resolution failure
   unless the resolver falls back to TCP.  It is known at this moment
   that limited number of security middle-box implementations support
   IPv6 fragments.

   A possible solution is to split a single DNS message across multiple
   UDP datagrams.  This DNS fragments mechanism is documented in
   [I-D.muks-dns-message-fragments] as an experimental IETF draft.

## 5. Experience on weird bugs
### 5.1 Root name compression issue
   [RFC1035]specifies DNS massage compression scheme which allows a
   domain name in a message to be represented as either: 1) a sequence
   of labels ending in a zero octet, 2) a pointer, 3) or a sequence of
   labels ending with a pointer.  It is designed to save more room of
   DNS packet.

   However in Yeti testbed, it is found that Knot 2.0 server compresses
   even the root.  It means in a DNS message the name of root (a zero
   octet) is replaced by a pointer of 2 octets.  As well, it is legal
   but breaks some tools (Go DNS lib in this bug report) which does not
   expect such name compression for root.  Both Knot and Go DNS lib have
   fixed that bug by now.

### 5.2 SOA update delay issue

   It is observed one server on Yeti testbed have some bugs on SOA
   update with more than 10 hours delay.  It is running on Bundy 1.2.0
   on FreeBSD 10.2-RELEASE.  A workaround is to check DM's SOA status in
   regular base.  But it still need some work to find the bug in code
   path to improve the software.



### 5.3 Renumbering Discussion

   With the recent renumbering of H root Server's IP address, there is a
   discussion of ways that resolvers can update their hint file.
   Traditional ways include using FTP protocol by doing a wget and using
   dig to double-check the servers' addresses manually.  Each way would
   depend on manual operation.  As a result, there are many old machines
   that have not updated their hint files.  As a proof, after completion
   of renumbering in thirteen years ago, there is an observation that
   the "Old J-Root" can still receive DNS query traffic
   [Renumbering-J-Root].

   This experiment proposal aims to find an automatic way for hint-file
   updating.  The already-completed work is a shell script tool which
   provides the function that updates a hint-file in file system
   automatically with DNSSEC and trust anchor validation.
   [Hintfile-Auto-Update]

   The methodology is straightforward.  The tool first queries the NS
   list for "." domain and queries A and AAAA records for every name on
   the NS list.  It requires DNSSEC validation for both the NS list and
   the A and AAAA answers.  After getting all the answers, the tool
   compares the new hint file with the old one.  If there is a
   difference, it renames the old one with a time-stamp and replaces the
   old one with the new one.  Otherwise the tool deletes the new hint
   file and nothing will be changed.

   Note that in current IANA root system the servers named in the root
   NS record are not signed.  So the tool can not fully work in the
   production network.  In Yeti root system some of the names listed in
   the NS record are signed, which provides a test environment for such
   a proposal.

## 6 .Conclusions and Recommendations
