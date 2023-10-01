---
stand_alone: true
ipr: trust200902
cat: std # Check
submissiontype: IETF
area: General [REPLACE]
wg: Internet Engineering Task Force

docname: draft-wu-savnet-inter-domain-architecture-04

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
  RFC6020:
  RFC6241:
  RFC5424:
informative:
  inter-domain-ps:
    target: https://datatracker.ietf.org/doc/draft-ietf-savnet-inter-domain-problem-statement/
    title: Source Address Validation in Inter-domain Networks Gap Analysis, Problem Statement, and Requirements 
    date: 2023
  RFC5635:
  RFC8955:

--- abstract

This document introduces an inter-domain SAVNET architecture, providing a comprehensive framework for guiding the design of inter-domain SAV mechanisms. The proposed architecture empowers ASes to establish SAV rules by sharing SAV-specific Information between themselves. During the incremental or partial deployment of SAV-specific Information, it can rely on general information, such as routing information from the RIB, to construct the SAV table when SAV-specific Information for an AS's prefixes is unavailable. Rather than delving into protocol extensions or implementations, this document primarily concentrates on proposing SAV-specific and general information and guiding how to utilize them to generate SAV rules. It also defines the architectural components and their relations.

--- middle

# Introduction

Attacks based on source IP address spoofing, such as reflective DDoS and flooding attacks, continue to present significant challenges to Internet security. Mitigating these attacks in inter-domain networks requires effective source address validation (SAV). While BCP84 {{RFC3704}} {{RFC8704}} offers some SAV solutions, such as ACL-based ingress filtering and uRPF-based mechanisms, existing inter-domain SAV mechanisms have limitations in terms of validation accuracy and operational overhead in different scenarios {{inter-domain-ps}}.

To address these issues, the inter-domain SAVNET architecture focuses on providing a comprehensive framework and guidelines for the design and implementation of new inter-domain SAV mechanisms. By proposing the SAV-specific Information which consists of prefixes of ASes and their corresponding legitimate incoming interfaces and is specialized for generating SAV rules, the inter-domain SAVNET architecture empowers ASes to generate accurate SAV rules. Meanwhile, a SAV-specific communication mechanism is used to define the data structure or format for communicating the SAV-specific Information, and the operations and timing for origination, processing, propagation, and termination of the messages which carry the SAV-specific Information, to achieve the delivery and automatic update of SAV-specific Information. Moreover, during the incremental/partial deployment period of the SAV-specific Information, the inter-domain SAVNET architecture can leverage the general information, such as the routing information from the RIB or the {prefix, maximum length, origin AS} information from the RPKI ROA Objects and the {AS, AS's Provider} information from the RPKI ASPA Objects, to generate the SAV rules when the SAV-specific Information is not available. To achieve this, the inter-domain SAVNET architecture assigns priorities to the SAV-specific Information and general information and generates the SAV rules based on priorities of the information in the SAV Information Base, and the SAV-specific Information has higher priority compared to the general information.

In addition, by defining the architectural components, relationships, and the SAV-specific Information and general information used in inter-domain SAV deployments, this document aims to promote consistency, interoperability, and collaboration among ASes. This document primarily describes a high-level architecture for consolidating SAV-specific Information and general information and deploying an inter-domain SAV mechanism between ASes. The document does not specify protocol extensions or implementations. Its purpose is to provide a conceptual framework and guidance for the development of inter-domain SAV mechanisms, allowing implementers to adapt and implement the architecture based on their specific requirements and network environments.

## Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

{:vspace}
SAV Rule:
: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: 
: The table or data structure that implements the SAV rules and is used for source address validation in the data plane. 

SAV-specific Information:
: The information consists of the source prefixes and their legitimate incoming interfaces to enter an AS.

SAV-related Information:
: The information is used to be consolidated to generate SAV rules and can be from SAV-specific Information or general information.

False Positive: 
: The validation results that the packets with legitimate source addresses are considered invalid improperly due to inaccurate SAV rules.

False Negative: 
: The validation results that the packets with spoofed source addresses are considered valid improperly due to inaccurate SAV rules.

SAV Information Base: 
: A table or data structure for storing SAV-related information collected from SAV-specific Information and general information.

# Design Goals

The inter-domain SAVNET architecture aims to improve SAV accuracy, facilitate partial deployment with low operational overhead, and develop a communication approach to communicate SAV-specific Information between ASes, while achieving efficient convergence and providing security guarantees to communicated information, which correspond to the requirements for new inter-domain SAV mechanisms {{inter-domain-ps}}. The overall goal can be broken down into the following aspects:

* First, the inter-domain SAVNET architecture should learn the real paths of source prefixes to any destination prefixes or permissible paths that can cover their real paths, and generate accurate SAV rules automatically based on the learned information to avoid false positives and reduce false negatives as much as possible.

* Second, the inter-domain SAVNET architecture should provide sufficient protection for the source prefixes of ASes that deploy it, even if only a portion of the Internet implements the architecture.

* Third, the inter-domain SAVNET architecture should adapt to dynamic networks and asymmetric routing scenarios automatically.

* Fourth, the inter-domain SAVNET architecture should communicate SAV-specific Information between ASes automatically with a communication approach.

* Fifth, the inter-domain SAVNET architecture should promptly detect the network changes and launch the convergence process quickly, while reducing false positives and false negatives during the convergence process.

* Last, the inter-domain SAVNET architecture should provide security guarantees for the communicated SAV-specific Information.

Other design goals, such as low operational overhead and easy implementation, are also very important and should be considered in specific protocols or protocol extensions.

# Inter-domain SAVNET Architecture 

~~~~~~~~~~
+-------------------------------------------------------+
|                       Other ASes                      |
+-------------------------------------------------------+
                                         | SAV-specific
                                         | Messages
+-------------------------------------------------------+                                       
|                                        |           AS |
|                                       \/              |
| +~~~~~~~~~~~~~~~~~~~~~+  +--------------------------+ |
| | General Information |  | SAV-specific Information | |
| +~~~~~~~~~~~~~~~~~~~~~+  +--------------------------+ |
|            |                           |              |
|           \/                          \/              |
| +---------------------------------------------------+ |
| | +-----------------------------------------------+ | |
| | |              SAV Information Base             | | |
| | +-----------------------------------------------+ | |
| |            SAV Information Base Manager           | |
| +---------------------------------------------------+ |
|                          |SAV Rules                   |
|                         \/                            |
|  +------------------------------------------------+   |
|  |                    SAV Table                   |   |
|  +------------------------------------------------+   |
+-------------------------------------------------------+
~~~~~~~~~~
{: #arch title="The inter-domain SAVNET architecture"}

{{arch}} shows the overview of the inter-domain SAVNET architecture. The inter-domain SAVNET architecture collects SAV-specific Information from the SAV-specific Messages of other ASes. The SAV-specific Information consists of the prefixes and their legitimate incoming interfaces to enter an AS. As a result, the SAV-specific Information can be used to generate SAV rules and build an accurate SAV table on each AS directly. In order to exchange SAV-specific Information between ASes, a new SAV-specific communication mechanism should be developed to carry the SAV-specific Information. Compared against existing inter-domain SAV mechanisms which rely on the general information such as routing information from the RIB, the SAV-specific Information can generate more accurate SAV rules, the root cause is that the SAV-specific Information is specially designed and communicated for inter-domain SAV, while the general information is not.

A SAV-specific communication mechanism should be developed to define the data structure or format for communicating the SAV-specific Information and the operations and timing for originating, processing, propagating, and terminating the messages which carry the information. Additionally, the SAV-specific Information will not be available for all ASes when the SAV-specific communication mechanism is on the incremental/partial deployment. Therefore, in the stage of incremental/partial deployment, the inter-domain SAVNET architecture can use the general information to generate SAV rules.

The SAV Information Base (SIB) can store the information from the SAV-specific Information and general information and is maintained by the SAV Information Base Manager (SIM), and then the SIM generates SAV rules based on the SIB and fills out the SAV table in the dataplane. The SIB can be managed by network operators using various methods such as YANG, Command-Line Interface (CLI), remote triggered black hole (RTBH) {{RFC5635}}, and Flowspec {{RFC8955}}.

Inter-domain SAVNET architecture does not prescribe any specific deployment models.

## SAV Information Base {#sib-sec}

The SIB is managed by the SAV Information Base Manager, which can consolidate SAV-related information from different sources. The SAV information sources of SIB include SAV-specific Information and general information, which are illustrated below:

* SAV-specific Information is the specifically collected information for SAV and exactly consists of the prefixes and their legitimate incoming interfaces to enter ASes.

* General information refers to the information that is not directly related to SAV but can be utilized to generate SAV rules, and includes routing information from the RIB or FIB, the {prefix, maximum length, origin AS} information from the RPKI ROA Objects, and the {AS, AS's Provider} information from the RPKI ASPA Objects.

In the future, if an information source is created but is not initially and specially used for SAV, the information can be categorized into general information. Therefore, the general information can be considered as the dual-use information.

~~~~~~~~~~
+---------------------------------------------------+----------+
|              SAV Information Sources              |Priorities|
+---------------------------------------------------+----------+
|              SAV-specific Information             |     1    |
+---------------------+-----------------------------+----------+
|                     | RPKI ROA Obj. and ASPA Obj. |     2    |
|                     +-----------------------------+----------+
| General Information |             RIB             |     3    |
|                     +-----------------------------+----------+
|                     |             FIB             |     4    |
+---------------------+-----------------------------+----------+
~~~~~~~~~~
{: #sav_src title="Priority ranking for the SAV information sources"}

{{sav_src}} presents a priority ranking for the SAV-specific Information and general information. SAV-specific Information has higher priority (i.e., 1) than the general information (i.e., 2), since the inter-domain SAVNET architecture uses the SAV-specific Information to carry more accurate information which comprises ASes' prefixes and their legitimate incoming interfaces. Therefore, once the SAV-specific Information for a prefix is available within the SIB, the inter-domain SAVNET generates SAV rules based on the information from the SAV-specific Information; otherwise, the inter-domain SAVNET generates SAV rules based on the general information. In other words, the inter-domain SAVNET architecture assigns priorities to the information from different SAV information sources, and always generates the SAV rules using the information with the highest priority.

~~~~~~~~~~
                           +------------------+
                           |     AS 3(P3)     |
                           +-+/\------+/\+----+
                              /          \
                             /            \ 
                            /              \
                     Itf.1 / (C2P)          \
                 +------------------+        \
                 |     AS 4(P4)     |         \
                 ++/\+---+/\+---+/\++          \
             Itf.2 /      | Itf.3  \ Itf.4      \
   P6[AS 1, AS 2] /       |         \            \
        P2[AS 2] /        |          \            \
                / (C2P)   |           \            \
+----------------+        |            \            \    
|    AS 2(P2)    |        | P1[AS 1]    \ P5[AS 5]   \ P5[AS 5]
+----------+/\+--+        | P6[AS 1]     \            \
             \            | NO_EXPORT     \            \
     P6[AS 1] \           |                \            \
      P1[AS 1] \          |                 \            \
      NO_EXPORT \ (C2P)   | (C2P)      (C2P) \      (C2P) \
              +----------------+           +----------------+
              |  AS 1(P1, P6)  |           |    AS 5(P5)    |
              +----------------+           +----------------+
~~~~~~~~~~
{: #as-topo title="An example of AS topology"}

~~~~~~~~~~
+-----+------+------------------+---------+------------------------+
|Index|Prefix|AS-level Interface|Direction| SAV Information Source |
+-----+------+------------------+---------+------------------------+
|  0  |  P3  |      Itf.1       |Provider |  General Information   | 
+-----+------+------------------+---------+------------------------+
|  1  |  P2  |      Itf.2       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  2  |  P1  |      Itf.2       |Customer |SAV-specific Information|
+-----+------+------------------+---------+------------------------+
|  3  |  P1  |      Itf.3       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  4  |  P6  |      Itf.2       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  5  |  P6  |      Itf.3       |Customer |SAV-specific Information|
|     |      |                  |         |  General Information   |
+-----+------+------------------+---------+------------------------+
|  6  |  P5  |      Itf.4       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  7  |  P5  |      Itf.1       |Provider |  General Information   |
+-----+------+------------------+---------+------------------------+
~~~~~~~~~~
{: #sib title="An example for the SAV information base in AS 4"}

We use the examples shown in {{as-topo}} and {{sib}} to introduce SIB and illustrate how to generate SAV rules based on the SIB. {{as-topo}} shows an example of AS topology and {{sib}} depicts an example of the SIB established in AS 4. As shown in {{as-topo}}, AS 4 has four AS-level interfaces, each connected to a different AS. Specifically, Itf.1 is connected to AS 3, Itf.2 to AS 2, Itf.3 to AS 1, and Itf.4 to AS 5. The arrows in the figure represent the commercial relationships between ASes. AS 3 is the provider of AS 4 and AS 5, while AS 4 is the provider of AS 1, AS 2, and AS 5, and AS 2 is the provider of AS 1. Assuming prefixes P1, P2, P3, P4, P5, and P6 are all the prefixes in the network. 

Each row of the SIB contains an index, prefix, AS-level incoming interface for the prefix, incoming direction, and the corresponding sources of these information. The incoming direction consists of customer, provider, and peer. For example, in {{sib}}, the row with index 0 indicates prefix P3's valid incoming interface is Itf.1, the ingress direction of P3 is AS 4's provider AS (AS 3), and these information is from the RIB. Note that the same SAV-related information may have multiple sources and the SIB records them all.

Recall that the inter-domain SAVNET architecture generates SAV rules based on the SAV-related information in the SIB and their priorities. In addition, in the case of an AS's provider/peer interfaces where loose SAV rules are applicable, the inter-domain SAVNET architecture recommends to use blocklist at such interfaces to only block the prefixes that are sure not to come at these interfaces, while in the case of an AS's customer interfaces that necessitate stricter SAV rules, the inter-domain SAVNET architecture recommends to use allowlist to only permit the prefixes that are allowed to come at these interfaces.

Based on the above rules, take the SIB in {{sib}} as an example to illustrate how the inter-domain SAVNET architecture generates the SAV table to perform SAV in the data plane. AS 4 can conduct SAV at its interfaces as follows: SAV at the interface Itf.1 blocks P1, P2, and P6 according to the rows with indexes 0, 1, 2, and 5 in the SIB, SAV at the interface Itf.2 permits P1 and P2 according to the rows with indexes 1 and 2 in the SIB, SAV at the interface Itf.3 permits P6 according to the row with index 5 in the SIB, and SAV at the interface Itf.4 permits P5 according to the row with index 6 in the SIB.

## SAV-specific Information

~~~~~~~~~~
+----------------------+              +----------------------+
|          AS          | SAV-specific |          AS          |
| +-----------------+  |  Messages    |  +-----------------+ |
| |  SAV-specific   |<-|--------------|->|   SAV-specific  | |
| |Message Processor|  |              |  |Message Processor| |
| +-----------------+  |              |  +-----------------+ |
+----------------------+              +----------------------+
~~~~~~~~~~
{: #sav_msg title="Exchanging SAV-specific Information with SAV-specific Messages between ASes"}

The SAV-specific Information is the information consisting of source prefixes and their legitimate incoming interfaces entering an AS, and the legitimate incoming interfaces are the interfaces where the packets whose source addresses are encompassed in the source prefixes come. Therefore, the SAV-specific Information can be expressed as &lt;Prefix, Interface&gt; pairs. It is noted that the same prefix may have different legitimate incoming interfaces for an AS, since the dataplane packets with the source addresses encompassed in the source prefixes may have different destination addresses.

The SAV-specific Information can be exchanged between ASes by the SAV-specific messages. As shown in {{sav_msg}}, the SAV-specific messages are used to propagate or originate the SAV-specific Information between ASes by the SAV-specific message processor. Within an AS, the SAV-specific message processor can obtain the next hop of the corresponding prefixes based on the local RIB and use SAV-specific messages to carry its own prefixes and/or the prefixes received from other ASes to the next hops for the corresponding destinations. When a SAV-specific processor receives the SAV-specific messages, it parses them to obtain the carried source prefixes, as well as the corresponding legitimate incoming interfaces by checking the interfaces which the SAV-specific messages arrive at. The SAV-specific message processor also checks the destination addresses of the SAV-specific messages, if the destination address of the SAV-specific message is itself, it will terminate the message, otherwise, it will forward it based on the local RIB. Following this, the SAV-specific messages can propagate the SAV-specific Information between ASes.

Moreover, if SAV-specific messages are used to exchange SAV-specific Information between ASes, a new SAV-specific communication mechanism would need to be developed to communicate the SAV-specific messages. The SAV-specific communication mechanism need to define the data structure or format to communicate the SAV-specific messages and the operations and timing for originating, processing, propagating, and terminating the messages. If an extension to an existing protocol is used to exchange SAV-specific Information, the corresponding existing protocol should not be affected. The SAV-specific message processor is the entity to support the SAV-specific communication mechanism. By parsing the SAV-specific messages, it obtains the ASN, the prefixes, the AS-level interfaces to receive the messages, and their incoming AS direction for maintaining the SIB. It is important to note that the SAV-specific message processor within an AS has the capability to establish connections with multiple SAV-specific message processors within different ASes, relying on either manual configurations by operators or an automatic mechanism.

The need for a SAV-specific communication mechanism arises from the facts that the SAV-specific Information needs to be obtained and communicated between ASes. Different from the general information such as routing information from the RIB, there are no existing mechanisms which can support the perception and communication of SAV-specific Information between ASes. Hence, a SAV-specific communication mechanism is needed to provide a medium and set of rules to establish communication between different ASes for the exchange of SAV-specific Information.

Additionally, the preferred AS paths of an AS may change over time due to route changes or network failures. The SAV-specific message processor should launch SAV-specific messages to adapt to the route changes in a timely manner.  The SAV-specific communication mechanism should handle route changes carefully to avoid false positives. The reasons for leading to false positives may include late detection of route changes, delayed message transmission, or packet losses. However, the detailed design of the SAV-specific communication mechanism for dealing with route changes is outside the scope of this document.

## SAV Information Base Manager

SAV Information Base Manager (SIM) consolidates SAV-related information from the SAV-specific Information and general information to initiate or update the SIB, while it generates SAV rules to populate the SAV table in the dataplane according to the SIB. The detailed collection methods of the SAV-related information depend on the deployment and implementation of the inter-domain SAV mechanisms and are out of scope for this document.

Using the SIB, SIM produces &lt;Prefix, Interface&gt; pairs to populate the SAV table, which represents the prefix and its legitimate incoming interface. It is worth noting that the interfaces in the SIB are logical AS-level interfaces and need to be mapped to the physical interfaces of the AS border routers.

## Management Channel and Information Channel

~~~~~~~~~~
+------+
|      |   Management Channel    +--------------------------------+ 
|      |<========================|        Network Operators       |
|      |                         +--------------------------------+
|      |
|      |   Information Channel   +--------------------------------+
|      |<------------------------| SAV-specific Message Processor |
|      |                         +--------------------------------+
| SIM  |
|      |   Information Channel   +--------------------------------+
|      |<------------------------|   RPKI ROA Obj. and ASPA Obj.  |
|      |                         +--------------------------------+
|      |
|      |   Information Channel   +--------------------------------+
|      |<------------------------|               RIB              |
|      |                         +--------------------------------+
+------+
~~~~~~~~~~
{: #sav_agent_config title="The management channel and information channel for collecting SAV-related information from different SAV information sources"}

The SAV-specific Information relies on the communication between SAV-specific message processors within ASes and the general information may be from multiple sources, such as the RIB and RPKI ROA objects and ASPA objects. Therefore, as illustrated in {{sav_agent_config}}, the SIM needs to receive the SAV-related information from these SAV information sources. We abstract the connections used to collect the SAV-related information from the sources as Information Channel. Also, the network operators can operate the SIB by manual configurations, such as YANG, CLI, RTBH {{RFC5635}}, and Flowspec {{RFC8955}}, where the approaches to implement these are abstracted as Management Channel.

The primary purpose of the management channel is to deliver manual configurations of network operators. Examples of such information include, but are not limited to: 

* SAV configurations using YANG, CLI, RTBH, or Flowspec.

* SAVNET operation and management.

* Inter-domain SAVNET provisioning.

Note that the information can be delivered at any time and requires reliable delivery for the management channel implementation.

The information channel serves as a means to transmit the SAV-specific Information and general information from various sources including the RIB and RPKI ROA objects and RPKI ASPA Objects. Additionally, it can carry telemetry information, such as metrics pertaining to forwarding performance, the count of spoofing packets and discarded packets, provided that the inter-domain SAVNET has access to such data. The information channel can include information regarding the prefixes associated with the spoofing traffic, as observed until the most recent time. 

# Partial/Incremental Deployment

The inter-domain SAVNET architecture MUST ensure support for partial/incremental deployment as it is not feasible to deploy it simultaneously in all ASes. The partial/incremental deployment of the inter-domain SAVNET architecture consists of different aspects, which include the partial/incremental deployment of the architecture and the partial/incremental deployment of the information sources. 

Within the architecture, the general information like the prefixes and topological information from RPKI ROA Objects and ASPA Objects and the routing information from the RIB can be obtained locally when the corresponding sources are available. Even when both SAV-specific Information and the information from RPKI ROA Objects and ASPA Objects are not available, the routing information from the RIB can be used to generate SAV rules.

Furthermore, it is not mandatory for all ASes to deploy SAV-specific message processors for SAV-specific Information. Instead, a SAV-specific message processor should be able to effortlessly establish a logical neighboring relationship with another AS that has deployed a SAV-specific message processor. The connections for communicating SAV-specific Information can be achieved by manual configurations set by operators or an automatic neighbor discovery mechanism. This flexibility enables the architecture to accommodate varying degrees of deployment, promoting interoperability and collaboration among participating ASes. During the partial/incremental deployment of SAV-specific message processor, the SAV-specific Information for the ASes which do not deploy SAV-specific message processor can not be obtained. To protect the prefixes of these ASes, inter-domain SAVNET architecture can use the SAV-related information from the general information in the SIB to generate SAV rules. At least, the routing information from the RIB can be always available in the SIB.

As more ASes adopt the inter-domain SAVNET architecture, the "deployed area" expands, thereby increasing the collective defense capability against source address spoofing. Furthermore, if multiple "deployed areas" can be logically interconnected across "non-deployed areas", these interconnected "deployed areas" can form a logical alliance, providing enhanced protection against address spoofing. Especially, along with more ASes deploy SAV-specific message processor and support the communication of SAV-specific Information, the generated SAV rules of the inter-domain SAVNET architecture to protect these ASes will become more accurate, as well as enhancing the protection capability against source address spoofing for the inter-domain SAVNET architecture.

In addition, releasing the SAV functions of the inter-domain SAVNET architecture incrementally is one potential way to reduce the deployment risks and can be considered in its deployment by network operators:

* First, the inter-domain SAVNET can only do the measurement in the data plane and do not take any other actions. Based on the measurement data, the operators can evaluate the effect of the inter-domain SAVNET on the legitimate traffic, including validation accuracy and forwarding performance, as well as the operational overhead.

* Second, the inter-domain SAVNET can open the function to limit the rate of the traffic that is justified as spoofing traffic. The operators can further evaluate the effect of the inter-domain SAVNET on the legitimate traffic and spoofing traffic, such as limiting the rate of all the spoofing traffic without hurting the legitimate traffic.

* Third, when the validation accuracy, forwarding performance, and operational overhead have been verified on a large scale by the live network, the inter-domain SAVNET can open the function to directly block the spoofing traffic that is justified by the SAV table in the data plane.

# Convergence Considerations

Convergence issues SHOULD be carefully considered in inter-domain SAV mechanisms due to the dynamic nature of the Internet. Internet routes undergo continuous changes, and SAV rules MUST proactively adapt to these changes, such as prefix and topology changes, in order to prevent false positives and reduce false negatives. To effectively track these changes, the SIM should promptly collect SAV-related information from various SAV information sources and consolidate them in a timely manner.

In particular, it is essential for the SAV-specific message processors to proactively communicate the changes of the SAV-specific Information between ASes and adapt to route changes promptly. However, during the routing convergence process, the traffic paths of the source prefixes can undergo rapid changes within a short period. The changes of the SAV-specific Information may not be communicated in time between ASes to update SAV rules, false positives or false negatives may happen. Such inaccurate validation is caused by the delays in communicating SAV-specific Information between ASes, which occur due to the factors like packet losses, unpredictable network latencies, or message processing latencies. The design of the SAV-specific communication mechanism should consider these issues to reduce the inaccurate validation.

Besides, for the inter-domain SAVNET architecture, the potential ways to deal with the inaccurate validation issues during the convergence of the SAV-specific communication mechanism is to consider using the information from RPKI ROA Objects and ASPA objects to generate SAV rules until the convergence process of the SAV-specific communication mechanism is finished, since these information is more stable and can help avoid false positives, and thus avoiding the impact to the legitimate traffic.

# Management Considerations

It is crucial to consider the operations and management aspects of SAV information sources, the SAV-specific communication mechanism, SIB, SIM, and SAV table in the inter-domain SAVNET architecture. The following guidelines should be followed for their effective management:

First, management interoperability should be supported across devices from different vendors or different releases of the same product, based on a unified data model such as YANG {{RFC6020}}. This is essential because the Internet comprises devices from various vendors and different product releases that coexist simultaneously.

Second, scalable operation and management methods such as NETCONF {{RFC6241}} and syslog protocol {{RFC5424}} should be supported. This is important as an AS may have hundreds or thousands of border routers that require efficient operation and management.

Third, management operations, including default initial configuration, alarm and exception reporting, logging, performance monitoring and reporting for the control plane and data plane, as well as debugging, should be designed and implemented in the protocols or protocol extensions. These operations can be performed either locally or remotely, based on the operational requirements.

By adhering to these rules, the management of SAV information sources and related components can be effectively carried out, ensuring interoperability, scalability, and efficient operations and management of the inter-domain SAVNET architecture.

# Security Considerations {#Security}

In the inter-domain SAVNET architecture, the SAV-specific message processor plays a crucial role in generating and disseminating SAV-specific Messages across different ASes. To safeguard against the potential risks posed by a malicious AS generating incorrect or forged SAV-specific Messages, it is important for the SAV-specific message processors to employ security authentication measures for each received SAV-specific Message. The majour security threats faced by inter-domain SAVNET can be categorized into two aspects: session security and content security. Session security pertains to verifying the identities of both parties involved in a session and ensuring the integrity of the session content. Content security, on the other hand, focuses on verifying the authenticity and reliability of the session content, thereby enabling the identification of forged SAV-specific Messages.

The threats to session security include:

* Session identity impersonation: This occurs when a malicious router deceitfully poses as a legitimate peer router to establish a session with the targeted router. By impersonating another router, the malicious entity can gain unauthorized access and potentially manipulate or disrupt the communication between the legitimate routers.

* Session integrity destruction: In this scenario, a malicious intermediate router situated between two peering routers intentionally tampers with or destroys the content of the relayed SAV-specific Message. By interfering with the integrity of the session content, the attacker can disrupt the reliable transmission of information, potentially leading to miscommunication or inaccurate SAV-related data being propagated.

The threats to content security include:

* Message alteration: A malicious router has the ability to manipulate or forge any portion of a SAV-specific Message. For example, the attacker may employ techniques such as using a spoofed Autonomous System Number (ASN) or modifying the AS Path information within the message. By tampering with the content, the attacker can potentially introduce inaccuracies or deceive the receiving ASes, compromising the integrity and reliability of the SAV-related information.

* Message injection: A malicious router injects a seemingly "legitimate" SAV-specific Message into the communication stream and directs it to the corresponding next-hop AS. This type of attack can be likened to a replay attack, where the attacker attempts to retransmit previously captured or fabricated messages to manipulate the behavior or decisions of the receiving ASes. The injected message may contain malicious instructions or false information, leading to incorrect SAV rule generation or improper validation.

* Path deviation: A malicious router intentionally diverts a SAV-specific Message to an incorrect next-hop AS, contrary to the expected path defined by the AS Path. By deviating from the intended routing path, the attacker can disrupt the proper dissemination of SAV-related information and introduce inconsistencies or conflicts in the validation process. This can undermine the effectiveness and accuracy of source address validation within the inter-domain SAVNET architecture.

Overall, inter-domain SAVNET shares similar security threats with BGP and can leverage existing BGP security mechanisms to enhance both session and content security. Session security can be enhanced by employing session authentication mechanisms used in BGP, such as MD5, TCP-AO, or Keychain. Similarly, content security can benefit from the deployment of existing BGP security mechanisms like RPKI, BGPsec, and ASPA. While these mechanisms can address content security threats, their widespread deployment is crucial. Until then, it is necessary to develop an independent security mechanism specifically designed for inter-domain SAVNET. One potential approach is for each origin AS to calculate a digital signature for each AS path and include these digital signatures within the SAV-specific Messages. Upon receiving a SAV-specific Message, the SAV-specific message processor can verify the digital signature to ascertain the message's authenticity. Furthermore, it is worth noting that the information channel of the inter-domain SAVNET architecture may need to operate over a network link that is currently under a source address spoofing attack. As a result, it may experience severe packet loss and high latency due to the ongoing attack, and the implementation of the information channel should ensure uninterrupted communication. Detailed security designs and considerations will be addressed in a separate draft, ensuring the robust security of inter-domain SAVNET.

# Privacy Considerations

TBD

# IANA Considerations {#IANA}

This document has no IANA requirements.


# Scope

In this architecture, the choice of protocols used for communication between the SIM and different SAV information sources is not limited. The inter-domain SAVNET architecture presents considerations on how to consolidate SAV-related information from various sources to generate SAV rules and perform SAV using the SAV table in the dataplane. The detailed design and implementation for SAV rule generation and SAV execution depend on the specific inter-domain SAV mechanisms employed.

This document does not cover administrative or business agreements that may be established between the involved inter-domain SAVNET parties. These considerations are beyond the scope of this document. However, it is assumed that authentication and authorization mechanisms can be implemented to ensure that only authorized ASes can communicate SAV-related information.

# Assumptions

This document makes the following assumptions:

* All ASes where the inter-domain SAVNET is deployed are assumed to provide the necessary connectivity between SAV-specific message processor and any intermediate network elements. However, the architecture does not impose any specific limitations on the form or nature of this connectivity. 

* Congestion and resource exhaustion can occur at various points in the inter-domain networks. Hence, in general, network conditions should be assumed to be hostile. The inter-domain SAVNET architecture must be capable of functioning reliably under all circumstances, including scenarios where the paths for delivering SAV-related information are severely impaired. It is crucial to design the inter-domain SAVNET system with a high level of resilience, particularly under extremely hostile network conditions. The architecture should ensure uninterrupted communication between inter-domain SAV-specific message processors, even when data-plane traffic saturates the link.

* The inter-domain SAVNET architecture does not impose rigid requirements for the SAV information sources that can be used to generate SAV rules. Similarly, it does not dictate strict rules on how to utilize the SAV-related information from diverse sources or perform SAV in the dataplane. Network operators have the flexibility to choose their approaches to generate SAV rules and perform SAV based on their specific requirements and preferences. Operators can either follow the recommendations outlined in the inter-domain SAVNET architecture or manually specify the rules for governing the use of SAV-related information, the generation of SAV rules, and the execution of SAV in the dataplane. 

* The inter-domain SAVNET architecture does not impose restrictions on the selection of the local AS with which AS to communicate SAV-specific Information. The ASes have the flexibility to establish connections for SAV-specific communication based on the manual configurations set by operators or other automatic mechanisms. 

* The inter-domain SAVNET architecture provides the flexibility to accommodate Quality-of-Service (QoS) policy agreements between SAVNET-enabled ASes or local QoS prioritization measures, but it does not make assumptions about their presence. These agreements or prioritization efforts are aimed at ensuring the reliable delivery of SAV-specific Information between SAV-specific message processors. It is important to note that QoS is considered as an operational consideration rather than a functional component of the inter-domain SAVNET architecture. 

* The information and management channels are loosely coupled and are used for collecting SAV-related information from different sources, and how the inter-domain SAVNET synchronize the management and operation configurations is out of scope of this document.


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

Many thanks to Alvaro Retana, Kotikalapudi Sriram, Rüdiger Volk, Xueyan Song, Ben Maddison, Jared Mauch, Joel Halpern, Aijun Wang, Jeffrey Haas, Xiangqing Chang, Changwang Lin, Mingxing Liu, Zhen Tan, etc. for their valuable comments on this document.