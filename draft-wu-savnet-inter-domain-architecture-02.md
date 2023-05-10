---
stand_alone: true
ipr: trust200902
cat: std # Check
submissiontype: IETF
area: General [REPLACE]
wg: Internet Engineering Task Force

docname: draft-wu-savnet-inter-domain-architecture-02

title: Inter-domain Source Address Validation (SAVNET) Architecture
abbrev: Inter-domain SAVNET Architecture
lang: en

author:
- ins: J. Wu
  name: Jianping Wu
  org: Tsinghua University
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: jianping@cernet.edu.cn
- ins: D. Li
  name: Dan Li
  org: Tsinghua University
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: tolidan@tsinghua.edu.cn
- ins: M. Huang
  name: Mingqing Huang
  org: Huawei
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: huangmingqing@huawei.com
- ins: L. Chen
  name: Li Chen
  org: Zhongguancun Laboratory
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: lichen@zgclab.edu.cn
- ins: N. Geng
  name: Nan Geng
  org: Huawei
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: gengnan@huawei.com
- ins: L. Liu
  name: Libin Liu
  org: Zhongguancun Laboratory
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: liulb@zgclab.edu.cn
- ins: L. Qin
  name: Lancheng Qin
  org: Tsinghua University
  city: Beijing
  country: China # use TLD (except UK) or country name
  email: qlc19@mails.tsinghua.edu.cn

normative:
  RFC8174:
  RFC3704:
  RFC8704:
  RFC5635:
  RFC8955:
  RFC4741:
  RFC5424:
informative:
  sav-table:
    target: https://datatracker.ietf.org/doc/draft-huang-savnet-sav-table/
    title: Source Address Validation Table Abstraction and Application
    date: 2023
  inter-domain-ps:
    target: https://datatracker.ietf.org/doc/draft-wu-savnet-inter-domain-problem-statement/
    title: Source Address Validation in Inter-domain Networks Gap Analysis, Problem Statement, and Requirements 
    date: 2023

--- abstract

Source address validation (SAV) is critical to preventing attacks based on source IP address spoofing. The proposed inter-domain SAVNET architecture enables an AS to generate SAV rules based on SAV-related information from various sources. This architecture integrates existing SAV mechanisms and offers ample design space for new inter-domain SAV mechanisms. The document focuses on architectural components and SAV-related information in an inter-domain SAVNET deployment, without specifying any protocols or protocol extensions.

--- middle

# Introduction

Attacks based on source IP address spoofing, such as reflective DDoS and flooding attacks, continue to pose significant challenges to Internet security. To mitigate such attacks in inter-domain networks, source address validation (SAV) is essential.
      
Although BCP84 {{RFC3704}} {{RFC8704}} provides some SAV solutions, such as uRPF-based mechanisms, the existing inter-domain SAV mechanisms have limitations in terms of validation accuracy and operational overhead {{inter-domain-ps}}. These mechanisms generate SAV rules based only on the Routing Information Base (RIB) of the local AS. In cases of asymmetric routing, the generated rules may not match the incoming direction of packets originating from a specific source address, leading to improper block or improper permit problems. Consequently, despite the deployment of SAV, an AS may still suffer from source address spoofing attacks from other ASes.

To address the limitations of existing inter-domain SAV mechanisms, this document proposes an inter-domain source address validation (SAVNET) architecture that provides a framework for developing new SAV mechanisms. The inter-domain SAVNET architecture enables an AS to generate precise SAV rules by leveraging SAV-related information from various categories of sources, including Manual Configurations, Routing Security Information (e.g., RPKI ROA Objects and ASPA Objects, and RIB), and SAV-tailored Messages from other ASes. These information consists of legitimate or nearly legitimate prefixes of the ASes and their corresponding incoming interfaces. By incorporating these information, the inter-domain SAVNET architecture enhances the accuracy of SAV rules beyond those generated solely based on the local RIB and ensure that the source addresses of the ASes providing the SAV-tailored information with SAV-tailored Messages are also protected.

This document focuses on high-level architecture designs and does not specify protocols or protocol extensions.

## Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

{:vspace}
SAV Rule:
: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: 
: The table or data structure that implements the SAV rules and is used for source address validation in the data plane. 

Real forwading paths: 
: The paths that the traffic go through in the data plane.

SAV-related information:
: The information is used to generate SAV rules and can be from various sources, such as manual configurations, SAV-related advertisements of ASes, RIB, etc.

SAV-tailored message: 
: The message is used to carry the SAV-related information between ASes and can be communicated following a dedicated SAV communication protocol.

Improper block: 
: The validation results that the packets with legitimate source addresses are blocked improperly due to inaccurate SAV rules.

Improper permit: 
: The validation results that the packets with spoofed source addresses are permitted improperly due to inaccurate SAV rules.

SIB: 
: SAV information base that is for storing SAV-related information collected from different information sources.

SMP: 
: SIB management protocol that represents any protocols for managing data in SIB.

# Design Goals

The inter-domain SAVNET architecture aims to improve SAV accuracy and facilitate partial deployment with low operational overhead. The overall goal can be broken down into the following aspects:

* First, the inter-domain SAVNET architecture should learn SAV-related information that consists of real forwarding paths or permissible paths of prefixes that can cover the real forwarding paths, and generate accurate and finer-grained SAV rules automatically based on the learned information to avoid improper block and reduce improper permit as much as possible.

* Second, the inter-domain SAVNET architecture should adapt to dynamic networks and asymmetric routing scenarios automatically.

* Third, the inter-domain SAVNET architecture should provide sufficient protection for the source prefixes of ASes that deploy it, even if only a portion of Internet implements the architecture.

* Last, the inter-domain SAVNET architecture should scale independently from the SAV information sources and the growth of the deployed Internet devices.

SAV-related information can be obtained from multiple sources, including local RIB, manual configurations, or SAV-tailored messages from other ASes. The inter-domain SAVNET architecture consolidates SAV-related information from multiple sources and generates SAV rules automatically, and adapts proactively to route changes in a timely manner to achieve the goals outlined above.

Other design goals, such as low operational overhead and easy implementation, are also very important and should be considered in specific protocols or protocol extensions and are out of scope for this document.

# Inter-domain SAVNET Architecture 

~~~~~~~~~~
                                         +----------------------+
                                         |      Other ASes      |
                                         +----------------------+
                                                     |SAV-tailored
                                                     |Messages
+-----------------------------------------------------------------+                                       
|    |YANG |CLI |SMP     |RIB |ROA |ASPA             |         AS |
|   \/    \/   \/       \/   \/   \/                \/            |
| +---------------+ +------------------+ +----------------------+ |
| |     Manual    | | Routing Security | |     SAV-tailored     | |
| | Configuration | |   Information    | |     Information      | |
| +---------------+ +------------------+ +----------------------+ |
|         |                   |                      |            |
|        \/                  \/                     \/            |
| +-------------------------------------------------------------+ |
| | +---------------------------------------------------------+ | |
| | |                   SAV Information Base                  | | |
| | +---------------------------------------------------------+ | |
| |                 SAV Information Base Manager                | |
| +-------------------------------------------------------------+ |
|                                |                                |
|                               \/                                |
|   +---------------------------------------------------------+   |
|   |                         SAV Table                       |   |
|   +---------------------------------------------------------+   |
+-----------------------------------------------------------------+
~~~~~~~~~~
{: #arch title="The inter-domain SAVNET Architecture"}

{{arch}} shows the overview of the inter-domain SAVNET architecture. The inter-domain SAVNET architecture collects SAV-related information from multiple sources, which can be categorized into three types: Manual Configuration, Routing Security Information, and SAV-tailored Information. The collected information is used to maintain the SAV Information Base (SIB). The SAV rules are generated based on the SIB and are used to fill out the SAV table in the dataplane.

Manual Configuration can be conducted by network operators using various methods such as YANG, Command-Line Interface (CLI), and SIB Management Protocol (SMP) like remote triggered black hole (RTBH) {{RFC5635}} and FlowSpec {{RFC8955}}. The Routing Security Information can be obtained within an AS and may come from RIB, Routing Information Messages, or RPKI ROA objects and ASPA objects. SAV-tailored Information, which contains real Forwarding paths of the prefixes, is transmitted using SAV-tailored Messages from other ASes. 

It is important to note that the inter-domain SAVNET architecture does not require all the information sources to generate SAV rules. All of the sources, including SAV-tailored Messages, are optional depending on their availability and the operational needs of the inter-domain SAV mechanisms. However, the SAV Information Base Manager (SIM) and SAV table are necessary components for generating SAV rules and performing SAV.

## SAV Information Base 

The SAV Information Base (SIB) is managed by the SAV Information Base Manager, which consolidates SAV-related information from various sources. Each row of the SIB contains an index, prefix, valid incoming interface for the prefix, incoming direction, and the corresponding sources of these SAV-related information.

~~~~~~~~~~
+-----+------+------------------+---------+---------------------------+
|Index|Prefix|AS-level Interface|Direction|   SAV Information Source  |
+-----+------+------------------+---------+---------------------------+
|  0  |  P1  |      Itf.1       |Provider | SAV-tailored Message, RIB | 
+-----+------+------------------+---------+---------------------------+
|  1  |  P1  |      Itf.2       |Provider |            RIB            |
+-----+------+------------------+---------+---------------------------+
|  2  |  P2  |      Itf.2       |Provider |    Manual Configuration   |
+-----+------+------------------+---------+---------------------------+
|  3  |  P3  |      Itf.3       |   Peer  |    SAV-tailored Message,  |
|     |      |                  |         |RPKI ROA Obj. and ASPA Obj.|
+-----+------+------------------+---------+---------------------------+
|  4  |  P4  |      Itf.4       |Customer |    SAV-tailored Message   |
+-----+------+------------------+---------+---------------------------+
|  5  |  P4  |      Itf.5       |Customer |RPKI ROA Obj. and ASPA Obj.|
+-----+------+------------------+---------+---------------------------+
|  6  |  P5  |      Itf.5       |Customer |            RIB            |
+-----+------+------------------+---------+---------------------------+
~~~~~~~~~~
{: #sib title="An Example for the SAV Information Base"}

~~~~~~~~~~
+------------+  (P2P)  +------------+
|  AS 1(P1)  +---------+  AS 2(P2)  |
+--------/\--+         +--/\--------+  
          \               /       
     (C2P) \             / (C2P)
      Itf.1 \           / Itf.2        
            +------------+   (P2P)   +------------+
            |    AS X    +-----------+  AS 3(P3)  |
            +-/\------/\-+ Itf.3     +------------+
         Itf.4 |       \ Itf.5
               |        \
               |         \ (C2P)
               |       +------------+
               |       |  AS 5(P5)  |
               |       +-/\---------+
               |         /
         (C2P) |        / (C2P)
             +------------+
             |  AS 4(P4)  |
             +------------+   
~~~~~~~~~~
{: #as-topo title="AS Topology"}

In order to provide a clear illustration of the SIB, {{sib}} depicts an example of an SIB established in AS X. As shown in {{as-topo}}, AS X has five interfaces, each connected to a different AS. Specifically, Itf.1 is connected to AS 1, Itf.2 to AS 2, Itf.3 to AS 3, Itf.4 to AS 4, and Itf.5 to AS 5. The arrows in the figure represent the commercial relationships between ASes, with AS 1 and AS 2 serving as providers for AS X, AS X as the lateral peer of AS 3, AS X as the provider for AS 4 and AS 5, and AS 5 as the provider for AS 4.

For example, in {{sib}}, the row with index 0 indicates prefix P1's valid incoming interface is Itf.1, the ingress direction of P1 is AS X's Provider AS, and these information is from the SAV-tailored Messages and RIB. Note that a SAV-related information may have multiple sources and the SIB records them all.
        
In sum, the SIB can be consolidated according to the SAV-related information from the following sources:

* Manual Configuaration: the SIB can obtain SAV-related configurations from the Manual Configurations of network operators using various methods, such as YANG, Command-Line Interface (CLI), and the SIM like RTBH {{RFC5635}} and FlowSpec {{RFC8955}}.

* SAV-tailored Message: the SIB can obtain real forwarding paths of the prefixes from the processed SAV-tailored Messages from other ASes.

* RPKI ROA Objects and ASPA Objects: the SIB can obtain the topological information from the RPKI ROA objects and ASPA objects over the local routing instance.

* Routing Information Base (RIB): the SIB can obtain the prefixes and their permissible routes including the optional ones and optional ones from the RIB.

~~~~~~~~~~
+---------------------------------------+--------+
|        SAV Information Source         |Priority|
+---------------------------------------+--------+
|         Manual Configuration          |    1   | 
+---------------------------------------+--------+
|         SAV-tailored Message          |    2   |
+---------------------------------------+--------+
|   RPKI ROA Objects and ASPA Objects   |    3   |
+---------------------------------------+--------+
|                  RIB                  |    4   |
+---------------------------------------+--------+
~~~~~~~~~~
{: #sav_src title="Priority Ranking for the SAV Information Sources"}

The SIB is maintained according to the SAV-related information from all the above information sources or part of them. It explicitly records the sources of the SAV-related information and can be compressed to reduce its size. Inter-domain SAVNET architecture allows operators to specify how to use the SAV-related information in the SIB by manual configurations. In fact, the SAV information sources also give indications about how to use the SAV-related information from them.
        
Notably, the SAV information sources are not equivalent, and finer-grained information sources, such as SAV-tailored Messages, can help generate more accurate SAV rules to reduce improper permit or avoid improper block. {{sav_src}} proposes the priority ranking for the SAV information sources in the SIB. Inter-domain SAVNET architecture can generate SAV rules based on the priorities of SAV information sources. For example, for the provider interfaces which can use loose SAV rules, inter-domain SAVNET may use the SAV-related information from all the sources (e.g., the rows with index 0, 1, and 2 in {{sib}}) to generate rules, while for the customer interfaces which need more strict SAV rules, inter-domain SAVNET architecture may only use the ones (e.g., the rows with index 4 and 6 in {{sib}}). Here, inter-domain SAVNET architecture uses the row with index 6 to generate SAV rules since only SAV-related information from the RIB is available for prefix P5. 

## SAV-tailored Messages

~~~~~~~~~~
+-----------------------+                       +-----------------------+
|          AS           |                       |          AS           |
| +------------------+  | SAV-tailored Messages |  +------------------+ |
| |   SAV-tailored   |--|-----------------------|->|   SAV-tailored   | |
| | Protocol Speaker |<-|-----------------------|--| Protocol Speaker | |
| +------------------+  | SAV-tailored Messages |  +------------------+ |
+-----------------------+                       +-----------------------+
~~~~~~~~~~
{: #sav_msg title="SAV-tailored Messages and SAV-tailored Protocol Speaker"}

As shown in {{sav_msg}}, SAV-tailored Messages are used to propagate or originate the real forwarding paths of prefixes between SAV-tailored Protocol Speakers. Within an AS, the SAV-tailored Protocol Speaker can obtain the next hop of the corresponding prefixes based on the routing table from the local RIB and use SAV-tailored Messages to carry the next hops of the corresponding prefixes and propagate them between ASes.

To generate the real forwarding paths of prefixes, the SAV-tailored Protocol Speaker connects to other SAV-tailored Protocol Speakers in other ASes, receives, processes, generates, or terminates SAV-tailored Messages. Then, it obtains the ASN, the prefixes, and their incoming AS direction for maintaining the SIB.

Moreover, the preferred AS paths of an AS may change over time due to route changes, including source prefix change and AS path change. The SAV-tailored Protocol Speaker should launch SAV-tailored Messages to adapt to the route changes in a timely manner. Inter-domain SAVNET should handle route changes carefully to avoid improper block. The reasons for this include late detection of route changes, delayed message transmission, or packet losses. However, the detailed design for dealing with route changes is outside the scope of this document.

## SAV Information Manager

SAV Information Manager (SIM) consolidates SAV-related information from multiple sources and generates SAV rules. It uses the SAV-related information to initiate or update the SIB and generates SAV rules to populate the SAV table according to the SIB. The collection methods of SAV-related information depend on the deployment and implementation of the inter-domain SAV mechanisms and are out of scope for this document.

Using the SIB, SIM produces <Prefixes, Interface> pairs to populate the SAV table, which represents the legitimate prefixes for each incoming interface. It's worth noting that the interfaces in the SIB are logical AS-level interfaces and need to be mapped to the physical interfaces of border routers within the corresponding AS.

~~~~~~~~~~
+------------------------------------+
| Source Prefix | Incoming Interface |
+---------------+--------------------+
|      P1       |          1         |
|      P2       |          2         |
|      P3       |          3         |
+------------------------------------+
~~~~~~~~~~
{: #sav_tab title="An Example of SAV table"}

{{sav_tab}} shows an example of the SAV table. The packets coming from other ASes will be validated by the SAV table. The router looks up each packet's source address in its local SAV table and gets one of three validity states: "Valid", "Invalid" or "Unknown". "Valid" means that there is a source prefix in SAV table covering the source address of the packet and the valid incoming interfaces covering the actual incoming interface of the packet. According to the SAV principle, "Valid" packets will be forwarded. "Invalid" means there is a source prefix in SAV table covering the source address, but the incoming interface of the packet does not match any valid incoming interface so that such packets will be dropped. "Unknown" means there is no source prefix in SAV table covering the source address. The packet with "unknown" addresses can be dropped or permitted, which depends on the choice of operators. The structure and usage of SAV table can refer to {{sav-table}}.

# Partial Deployment

Inter-domain SAVNET architecture must support partial deployment since it is impossible to deploy it in all ASes simultaneously. In the inter-domain SAVNET architecture, manual configuration, topological information, and known prefixes can be obtained locally. Even for the real forwarding path information, it is not necessary for all ASes to deploy the SAV-tailored Protocol Speakers. A SAV-tailored Protocol Speaker can easily establish a logical neighboring relationship with another AS that deploys a SAV-tailored Protocol Speaker.

Overall, ASes which deploys inter-domain SAVNET architecture cannot spoof each other, and non-deployed ASes cannot send spoofed packets to the deployed ASes by forging their source addresses. With the merger of different ASes where the inter-domain SAVNET architecture is deployed, the "deployed area" will gradually expand, and the defense capability against source address spoofing will gradually increase. Moreover, if multiple "deployed areas" can be logically connected (by crossing the "non-deployed areas"), these "deployed areas" can form a logic alliance to protect their addresses from being forged.

# Convergence Considerations

Convergence issues SHOULD be considered in the inter-domain SAV mechanisms since the Internet is a consistently changing system. internet routes change over time, and SAV rules need to proactively adapt to the these changes, including prefix and topology changes, to avoid improper block or reduce improper permit. To track these changes, the SIM SHOULD collect and consolidate the SAV-related information from SAV information sources in a timely manner.

For SAV-related information from Manual Configurations, the SIM can receive the configurations directly from network operators and generate the corresponding SAV rules in a timely manner.

For the Routing Security Information, the mechanism can rely on the convergence mechanism of routing protocols, such as BGP. However, inaccurate validation may occur before the route convergence process is finished, which is also a problem with existing uRPF-based mechanisms. This may be acceptable since the real forwarding paths of prefixes may not be consistent during the convergence process.

For the SAV-tailored information, the SAV-tailored Protocol Speaker SHOULD launch SAV-tailored Messages to adapt to the route changes in a timely manner. Note that before the route convergence process is completed, the SAV rules should be generated automatically according to the Routing Security Information, since the real forwarding paths easily change in a short time and the temporary SAV-tailored information may lead to improper block.

# Management Considerations

It is important to consider the operations and management of various SAV information sources, the SAV-tailored protocol, SIB, SIM, and SAV table. The following rules should be followed for their management:

* First, management interoperability SHOULD be supported across devices from different vendors or across different releases of the same product based on a unified data model, such as YANG. This is necessary since devices from different vendors and different releases of the same vendor product exist simultaneously in the Internet.

* Second, scalable operation and management methods, such as NETCONF {{RFC4741}} and syslog protocol {{RFC5424}}, SHOULD be supported since an AS may have hundreds or thousands of border routers to be operated and managed.

* Third, management operations, including default initial configuration, alarm and exception reporting, logging, performance monitoring and reporting about the control plane and data plane, and debugging, SHOULD be designed and implemented in the protocols or protocol extensions. These operations can be performed locally or remotely, based on the operational requirements.

# Security Considerations {#Security}

In inter-domain SAVNET architecture, the SAV-tailored Protocol Speaker generates and propagates SAV-tailored Messages among different ASes. To prevent a malicious AS from generating incorrect or forged SAV-tailored Messages, the SAV-tailored Protocol Speakers need to perform security authentication on each received SAV-tailored Message. Security threats to inter-domain SAVNET come from two aspects: session security and content security. Session security means that the identities of both parties in a session can be verified and the integrity of the session content can be ensured. Content security means that the authenticity and reliability of the session content can be verified, i.e., the forged SAV-tailored Message can be identified.

The threats to session security include:

* Session identity impersonation: A malicious router masquerades as a peer of another router to establish a session with the router.

* Session integrity destroying: A malicious intermediate router between two peering routers destroys the content of the relayed SAV-tailored Message.

The threats to content security include:

* Message alteration: A malicious router forges any part of a SAV-tailored Message, for example, using a spoofed ASN or AS Path.

* Message injection: A malicious router injects a "legitimate" SAV-tailored Message and sends it to the corresponding next-hop AS, such as replay attacks.

* Path deviation: A malicious router sends a SAV-tailored Message to a wrong next-hop AS which conflicts with the AS Path.

Overall, inter-domain SAVNET faces security threats similar to those of BGP. To enhance session security, inter-domain SAVNET may use the same session authentication mechanisms as BGP, i.e., performing session authentication based on MD5, TCP-AO, or Keychain. To enhance content security, existing BGP security mechanisms (e.g., RPKI, BGPsec, and ASPA) can also help address content security threats to inter-domain SAVNET. But until these mechanisms are widely deployed, an independent security mechanism for inter-domain SAVNET is needed. In the preliminary idea, each origin AS calculates a digital signature for each AS path and adds these digital signatures into the SAV-tailored Message. When a SAV-tailored Protocol Speaker receives a SAV-tailored Message, it can check the digital signature to identify the authenticity of this message. Specific security designs will be included in a separate draft.

# Privacy Considerations

This document should not affect the privacy of the Internet.

# IANA Considerations {#IANA}

This document has no IANA requirements.

# Contributors

Igor Lubashev  
  Akamai Technologies  
  145 Broadway  
  Cambridge, MA, 02142  
  United States of America  
  Email: ilubashe@akamai.com

Many thanks to Igor Lubashev for the significantly helpful revision suggestions.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}

Many thanks to Alvaro Retana, Rüdiger Volk, Xueyan Song, Ben Maddison, Jared Mauch, Joel Halpern, Aijun Wang, Jeffrey Haas, Fang Gao, Mingxing Liu, Zhen Tan, etc. for their valuable comments on this document.