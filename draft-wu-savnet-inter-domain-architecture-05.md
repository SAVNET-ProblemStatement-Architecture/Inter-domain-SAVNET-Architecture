---
stand_alone: true
ipr: trust200902
cat: std # Check
submissiontype: IETF
area: General [REPLACE]
wg: Internet Engineering Task Force

docname: draft-wu-savnet-inter-domain-architecture-05

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
  intra-domain-arch:
    target: https://datatracker.ietf.org/doc/draft-li-savnet-intra-domain-architecture/
    title: Intra-domain Source Address Validation (SAVNET) Architecture
    date: 2023
  RFC5635:
  RFC8955:
  RFC8210:
  manrs:
    target: https://www.manrs.org/netops/guide/antispoofing/
    title: MANRS Implementation Guide
    author:
     - org: MANRS
    date: 2023
  nist:
    target: https://www.nist.gov/publications/resilient-interdomain-traffic-exchange-bgp-security-and-ddos-mitigation
    title: "Resilient Interdomain Traffic Exchange: BGP Security and DDos Mitigation"
    author:
     - org: NIST
    date: 2019

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
: The table or data structure that implements the SAV rules and is used for source address validation on the data plane.

Local Routing Information:
: The information is stored in ASBR's local RIB or FIB and can be used to generate SAV rules in addition to the routing purpose. 

SAV-specific Information:
: The information is specialized for SAV rule generation, includes the source prefixes and their legitimate incoming interfaces to enter an AS, and is communicated between ASes.

General Information:
: The information is not specialized for SAV but can be utilized to generate SAV rules, and is initially utilized for other purposes. For example, the local routing information is one kind of general information.

SAV-related Information:
: The information is used to be consolidated to generate SAV rules and can be from SAV-specific Information or general information.

SAV-specific Communication Mechanism:
: The mechanism is used to communicate SAV-specific information between ASes, and it can be a new protocol or an extension to an existing protocol.

SAV Information Base: 
: A table or data structure for storing SAV-related information collected from SAV-specific Information and general information.

False Positive: 
: The validation results that the packets with legitimate source addresses are considered invalid improperly due to inaccurate SAV rules. False positive may induce improper block problems if routers block the "invalid" packets.

False Negative: 
: The validation results that the packets with spoofed source addresses are considered valid improperly due to inaccurate SAV rules. False negative may induce improper permit problems if routers accept the "valid" packets. 

# Design Goals {#goal-sec}

The inter-domain SAVNET architecture aims to improve SAV accuracy, facilitate partial deployment with low operational overhead, and develop a communication approach to communicate SAV-specific Information between ASes, while achieving efficient convergence and providing security guarantees to communicated information, which correspond to the requirements for new inter-domain SAV mechanisms {{inter-domain-ps}}. The overall goal can be broken down into the following aspects:

* G1: The inter-domain SAVNET architecture should learn the real paths of source prefixes to any destination prefixes or permissible paths that can cover their real paths, and generate accurate SAV rules automatically based on the learned information to avoid false positives and reduce false negatives as much as possible.

* G2: The inter-domain SAVNET architecture should provide sufficient protection for the source prefixes of ASes that deploy it, even if only a portion of the Internet implements the architecture.

* G3: The inter-domain SAVNET architecture should adapt to dynamic networks and asymmetric routing scenarios automatically.

* G4: The inter-domain SAVNET architecture should communicate SAV-specific Information between ASes automatically with a communication approach.

* G5: The inter-domain SAVNET architecture should promptly detect the network changes and launch the convergence process quickly, while reducing false positives and false negatives during the convergence process.

* G6: The inter-domain SAVNET architecture should provide security guarantees for the communicated SAV-specific Information.

Other design goals, such as low operational overhead and easy implementation, are also very important and should be considered in specific protocols or protocol extensions.

# Inter-domain SAVNET Architecture Overview

~~~~~~~~~~
+-----------------------------------------------------------+
|                          Other ASes                       |
+-----------------------------------------------------------+
                                           | SAV-specific
                                           | Messages
+-----------------------------------------------------------+
|                                          |                |
| +-------------------------------------------------------+ |                                      
| |                        SAV Agent       |              | |
| |                                       \/              | |
| | +---------------------+  +--------------------------+ | |
| | | General Information |  | SAV-specific Information | | |
| | +---------------------+  +--------------------------+ | |
| |            |                           |              | |
| |           \/                          \/              | |
| | +---------------------------------------------------+ | |
| | | +-----------------------------------------------+ | | |
| | | |              SAV Information Base             | | | |
| | | +-----------------------------------------------+ | | |
| | |            SAV Information Base Manager           | | |
| | +---------------------------------------------------+ | |
| |                          |SAV Rules                   | |
| +-------------------------------------------------------+ |
|                            |                              |
|                           \/                              |
| +-------------------------------------------------------+ |
| |                      SAV Table                        | |
| +-------------------------------------------------------+ |
+-----------------------------------------------------------+
~~~~~~~~~~
{: #arch title="The inter-domain SAVNET architecture"}

The inter-domain SAVNET architecture depicted in {{arch}} collects SAV-specific information from the SAV-specific messages of other ASes. The SAV-specific information consists of the prefixes and their legitimate incoming interfaces of the ASes. As a result, the SAV-specific information can be used to generate SAV rules and build an accurate SAV table on each AS directly (G1). When the SAV-specific information is not available due to incremental/partial deployment, the inter-domain SAVNET architecture can also leverage the general information such as the routing information from the RIB to generate SAV rules (G2). 

The inter-domain SAVNET architecture prioritizes the SAV-specific information and general information. When both SAV-specific information and general information are available, it guides to adopt the SAV-specific information to generate SAV rules. The SAV-specific information can help generate more accurate SAV rules, the rationale is that the SAV-specific information is designed specifically for inter-domain SAV to carry the prefixes and their legitimate incoming interfaces (G1).

The SAV Agent should launch SAV-specific messages to adapt to the route changes in a timely manner (G3). The SAV-specific communication mechanism should handle route changes carefully to avoid false positives. The reasons for leading to false positives may include late detection of route changes, delayed message transmission, or packet losses. During the convergence process of the SAV-specific communication mechanism, the inter-domain SAVNET architecture can use the information from RPKI ROA Objects and ASPA objects to generate SAV rules until the convergence process is finished, since these information is more stable and can help avoid false positives, and thus avoiding the impact to the legitimate traffic (G5). However, the detailed design of the SAV-specific communication mechanism for dealing with route changes is outside the scope of this document.

A new SAV-specific information communication mechanism is required to exchange SAV-specific information between ASes. It should define the data structure or format for communicating the SAV-specific information and the operations and timing for originating, processing, propagating, and terminating the messages which carry the information (G4). 

Besides, regarding the security concerns, the inter-domain SAVNET architecture shares similar security threats with BGP and can leverage existing BGP security mechanisms to enhance both session and content security (G6).

Moreover, the SAV Information Base (SIB) can store SAV-specific and general information and is maintained by the SAV Information Base Manager (SIM). The SIM generates SAV rules based on the SIB and fills out the SAV table in the data plane. The SIB can be managed by network operators using various methods such as YANG, Command-Line Interface (CLI), remote triggered black hole (RTBH), and Flowspec. The detailed collection methods of the SAV-related information depend on the deployment and implementation of the inter-domain SAV mechanisms and are out of scope for this document.

It is worth noting that the interfaces in the SIB are logical AS-level interfaces and need to be mapped to the physical interfaces of the AS border routers.

# SAV Information Base {#sib-sec}

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
Priority ranking from 1 to 4 represents high to low priority.
~~~~~~~~~~
{: #sav_src title="Priority ranking for the SAV information sources."}

{{sav_src}} presents the priority ranking for the SAV-specific Information and general information. Priority ranking from 1 to 4 represents high to low priority. Inter-domain SAVNET architecture should use the SAV information based on the priorities. Once the SAV-specific Information for a prefix is available within the SIB, the inter-domain SAVNET generates SAV rules based on the information from the SAV-specific Information; otherwise, the inter-domain SAVNET generates SAV rules based on the general information. In other words, the inter-domain SAVNET architecture assigns priorities to the information from different SAV information sources, and always generates the SAV rules using the information with the highest priority, as long as the information is available.

The priority ranking recommendation for different SAV information sources in {{sav_src}} is based on the accuracy of SAV rules generated based on the information from the different sources. That is avoiding improper blocks and minimizing improper permits. SAV-specific Information has higher priority than the general information, since the SAV-specific Information is specifically designed to carry more accurate SAV information which comprises ASes' prefixes and their legitimate incoming interfaces to an AS. The general information from RPKI ROA Objects and ASPA Objects, RIB, and FIB has different priorities, ranking 2, 3, and 4, respectively. The information from RPKI ROA Object and ASPA Object has higher priority than the one from RIB and FIB, this is because RPKI ROA Object and ASPA Object can provide authoritative prefixes and topology information, which can be used to generate more accurate SAV rules. The information from RPKI ROA Object and ASPA Object is more stable and can be used to reduce the risk of improper blocks during the convergence process of the network. Although the fundamental information source for RIB and FIB is the same, the RIB consists of more back path information than the FIB, which can reduce improper blocks.

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
Both AS 1 and AS 4 deploy the inter-domain SAVNET architecture 
and can exchange the SAV-specific information with each other, 
while other ASes do not deploy it.
~~~~~~~~~~
{: #as-topo title="An example of AS topology"}

~~~~~~~~~~
+-----+------+------------------+---------+------------------------+
|Index|Prefix|AS-level Interface|Direction| SAV Information Source |
+-----+------+------------------+---------+------------------------+
|  0  |  P1  |      Itf.2       |Customer |SAV-specific Information| 
+-----+------+------------------+---------+------------------------+
|  1  |  P1  |      Itf.3       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  2  |  P2  |      Itf.2       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  3  |  P3  |      Itf.1       |Provider |  General Information   |
+-----+------+------------------+---------+------------------------+
|  4  |  P5  |      Itf.1       |Provider |  General Information   |
+-----+------+------------------+---------+------------------------+
|  5  |  P5  |      Itf.4       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
|  6  |  P6  |      Itf.2       |Customer |  General Information   |
|     |      |                  |         |SAV-specific Information|
+-----+------+------------------+---------+------------------------+
|  7  |  P6  |      Itf.3       |Customer |  General Information   |
+-----+------+------------------+---------+------------------------+
~~~~~~~~~~
{: #sib title="An example for the SAV information base in AS 4"}

We use the examples shown in {{as-topo}} and {{sib}} to introduce SIB and illustrate how to generate SAV rules based on the SIB. {{as-topo}} shows an example of AS topology and {{sib}} depicts an example of the SIB established in AS 4. As shown in {{as-topo}}, AS 4 has four AS-level interfaces, each connected to a different AS. Specifically, Itf.1 is connected to AS 3, Itf.2 to AS 2, Itf.3 to AS 1, and Itf.4 to AS 5. The arrows in the figure represent the commercial relationships between ASes. AS 3 is the provider of AS 4 and AS 5, while AS 4 is the provider of AS 1, AS 2, and AS 5, and AS 2 is the provider of AS 1. Assuming prefixes P1, P2, P3, P4, P5, and P6 are all the prefixes in the network. 

Each row of the SIB contains an index, prefix, AS-level incoming interface for the prefix, incoming direction, and the corresponding sources of this information. The incoming direction consists of customer, provider, and peer. For example, in {{sib}}, the row with index 0 indicates prefix P1's valid incoming interface is Itf.2, the ingress direction of P1 is AS 4's customer AS (AS 2), and this information is from the RIB. Note that the same SAV-related information may have multiple sources and the SIB records them all, such as the row indexed 6 in {{sib}}. Moreover, the SIB should be carefully implemented in the specific protocol or protocol extensions to avoid becoming a heavy burden of the router, and the similar optimization approaches used for the RIB may be applied.

Recall that the inter-domain SAVNET architecture generates SAV rules based on the SAV-related information in the SIB and their priorities. In addition, in the case of an AS's provider/peer interfaces where loose SAV rules are applicable, the inter-domain SAVNET architecture recommends to use blocklist at such interfaces to only block the prefixes that are sure not to come at these interfaces, while in the case of an AS's customer interfaces that necessitate stricter SAV rules, the inter-domain SAVNET architecture recommends to use allowlist to only permit the prefixes that are allowed to come at these interfaces.

Based on the above rules, take the SIB in {{sib}} as an example to illustrate how the inter-domain SAVNET architecture generates the SAV table to perform SAV in the data plane. AS 4 can conduct SAV at its interfaces as follows: SAV at the interface Itf.1 blocks P1, P2, and P6 according to the rows indexed 0, 2, and 6 in the SIB, SAV at the interface Itf.2 permits P1, P2, and P6 according to the rows indexed 0, 2, and 6 in the SIB, SAV at the interface Itf.3 does not permit any prefixes according to the row indexed 0, 1, 6, and 7 in the SIB, and SAV at the interface Itf.4 permits P5 according to the row indexed 6 in the SIB.

# SAVNET Communication Channel

~~~~~~~~~~
+------+
|      |  SAV-specific Information Channel  +-----------------------------+ 
|      |<===================================|      SAV Agent in AS 2      |
|      |                                    +-----------------------------+
|      |
|      |     General Information Channel    +-----------------------------+
|      |<-----------------------------------| RPKI ROA Obj. and ASPA Obj. |
|      |                                    +-----------------------------+
| SAV  |
|Agent |     General Information Channel    +-----------------------------+
|  in  |<-----------------------------------|             RIB             |
| AS 1 |                                    +-----------------------------+
|      |
|      |     General Information Channel    +-----------------------------+
|      |<-----------------------------------|             FIB             |
|      |                                    +-----------------------------+
|      |
|      |        Management Channel          +-----------------------------+
|      |<-----------------------------------|      Network Operators      |
|      |                                    +-----------------------------+
+------+
~~~~~~~~~~
{: #sav_agent_config title="The management channel and information channel for collecting SAV-related information from different SAV information sources"}

The SAV-specific Information relies on the communication between SAV Agents within ASes and the general information can be from multiple sources, such as RPKI ROA objects and ASPA objects, RIB, and FIB. Therefore, as illustrated in {{sav_agent_config}}, the SAV Agent needs to receive the SAV-related information from these SAV information sources. Besides, the SAV Agent also needs to accept the configurations from network operators for the management operations. The connections for collecting these types of information are abstracted as SAVNET communication channel, which includes SAV-specific information channel, general information channel, and management channel. 

## SAV-specific Information Channel

~~~~~~~~~~
+----------------------+                          +----------------------+
|      AS 1 (P1)       |   SAV-specific Messages  |       AS 2 (P2)      |
| +-----------------+  |       (P1, Itf.2)  Itf.2 |  +-----------------+ |
| |       SAV       |--|--------------------------|->|       SAV       | |
| |      Agent      |<-|--------------------------|--|      Agent      | |
| +-----------------+  | Itf.1 (P2, Itf.1)        |  +-----------------+ |
+----------------------+   SAV-specific Messages  +----------------------+
~~~~~~~~~~
{: #sav_msg title="An example for Exchanging SAV-specific Information with SAV-specific Messages between AS1 and AS 2"}

The SAV-specific information channel is the abstraction of the connections between ASes for communicating SAV-specific information. Each AS pair which deploys inter-domain SAVNET architecture has a SAV-specific information channel between. An AS can have multiple SAV-specific information channels with others. Within the SAV-specific information channel, the SAV-specific message is used to carry the SAV-specific information.

{{sav_msg}} uses an example for exchanging SAV-specific information with SAV-specific messages between AS1 and AS 2. The SAV-specific Information is the information consisting of source prefixes and their legitimate incoming interfaces entering an AS, and the legitimate incoming interfaces are the interfaces where the packets whose source addresses are encompassed in the source prefixes come. Therefore, the SAV-specific Information can be expressed as &lt;Prefix, Interface&gt; pairs, e.g., (P1, Itf.2) and (P2, Itf.1) in {{sav_msg}}. It is noted that the same prefix may have different legitimate incoming interfaces for an AS, since the dataplane packets with the source addresses encompassed in the source prefixes may have different destination addresses.

The SAV-specific Information can be exchanged between ASes by the SAV-specific messages. As shown in {{sav_msg}}, the SAV-specific messages are used to propagate or originate the SAV-specific Information between ASes by the SAV Agent. For an AS which initiates its own SAV-specific messages, the SAV Agent within the AS can obtain the next hop of the corresponding prefixes based on the local RIB and use SAV-specific messages to carry the AS's prefixes to the next hops for the corresponding destinations. This is achieved by setting the SAV-specific messages' destination addresses as the destination addresses in the local RIB. When the SAV Agents of other ASes receive the SAV-specific messages, they parse the messages to obtain the carried source prefixes, as well as the corresponding legitimate incoming interfaces by checking the interfaces which the SAV-specific messages arrive at. Meanwhile, the SAV Agents also check the destination addresses of the SAV-specific messages, if their destination addresses are ASes themselves, they will terminate the message; otherwise, they will forward them based on the local RIB. Following this, the SAV-specific messages can propagate the SAV-specific Information between ASes.

Moreover, if SAV-specific messages are used to exchange SAV-specific Information between ASes, a new SAV-specific communication mechanism would need to be developed to communicate the SAV-specific messages. The SAV-specific communication mechanism needs to define the data structure or format to communicate the SAV-specific messages and the operations and timing for originating, processing, propagating, and terminating the messages. If an extension to an existing protocol is used to exchange SAV-specific Information, the corresponding existing protocol should not be affected. The SAV Agent is the entity to support the SAV-specific communication mechanism. By parsing the SAV-specific messages, it obtains the ASN, the prefixes, the AS-level interfaces to receive the messages, and their incoming AS direction for maintaining the SIB. It is important to note that the SAV Agent within an AS has the capability to establish connections with multiple SAV Agents within different ASes, relying on either manual configurations by operators or an automatic mechanism.

The need for a SAV-specific communication mechanism arises from the facts that the SAV-specific Information needs to be obtained and communicated between ASes. Different from the general information such as routing information from the RIB, there are no existing mechanisms which can support the perception and communication of SAV-specific Information between ASes. Hence, a SAV-specific communication mechanism is needed to provide a medium and set of rules to establish communication between different ASes for the exchange of SAV-specific Information.

Furthermore, an AS which initiates an SAV-specific message needs to assemble its source prefixes into the SAV-specific messages. In order to obtain all the source prefixes of an AS, the inter-domain SAVNET architecture can communicate with the intra-domain SAVNET architecture {{intra-domain-arch}} to obtain all the prefixes belonging to the AS.

Additionally, the preferred AS paths of an AS may change over time due to route changes or network failures. The SAV Agent should launch SAV-specific messages to adapt to the route changes in a timely manner.  The SAV-specific communication mechanism should handle route changes carefully to avoid false positives. The reasons for leading to false positives may include late detection of route changes, delayed message transmission, or packet losses. However, the detailed design of the SAV-specific communication mechanism for dealing with route changes is outside the scope of this document.

## General Information Channel

The general information channel is the abstraction of the connections between ASes for communicating routing information or the connections between AS and RPKI cache servers for obtaining {prefix, maximum length, origin AS} and {AS, AS's Provider} information. The communication of the general information within the general information channel can rely on existing protocols, such as BGP and RTR {{RFC8210}}.


## Management Channel

The management channel is the abstraction of the connections between SAV Agent and network operators. The primary purpose of the management channel is to deliver manual configurations of network operators. Examples of the management configurations include, but are not limited to: 

* SAV configurations using YANG, CLI, RTBH, or Flowspec.

* SAVNET operation and management.

* Inter-domain SAVNET provisioning.

Note that the configuration information can be delivered at any time and requires reliable delivery for the management channel implementation. Additionally, the management channel can carry telemetry information, such as metrics pertaining to forwarding performance, the count of spoofing packets and discarded packets, provided that the inter-domain SAVNET has access to such data. It can include information regarding the prefixes associated with the spoofing traffic, as observed until the most recent time. 

# Use Cases

This section utilizes the sample use cases to showcase that the inter-domain SAVNET architecture can improve the validation accuracy in the scenarios of limited propagation of prefixes, hidden prefixes, reflection attacks, and direct attacks, compared to existing SAV mechanisms, which are also utilized for the gap analysis of existing inter-domain SAV mechanisms in {{inter-domain-ps}}. In the following, these use cases are discussed for SAV at customer interfaces and SAV at provider/peer interfaces, respectively.

## SAV at Customer Interfaces

In order to prevent the source address spoofing, operators can enable ACL-based ingress filtering, source-based RTBH filtering, and/or uRPF-based mechanisms at customer interfaces, namely Strict uRPF, FP-uRPF, VRF uRPF, or EFP-uRPF {{manrs}} {{nist}}. However, as analyzed in {{inter-domain-ps}}, uRPF-based mechanisms may lead to false positives in two inter-domain scenarios: limited propagation of prefixes and hidden prefixes, or may lead to false negatives in the scenarios of source address spoofing attacks within a customer cone, while ACL-based ingress filtering and source-based RTBH filtering need to update SAV rules in a timely manner and lead to high operational overhead. The following showcases that the inter-domain SAVNET architecture can avoid false positives and false negatives in these scenarios.

### Limited Propagation of Prefixes

~~~~~~~~~~
                          +----------------+
                          |    AS 3(P3)    |
                          +-+/\----+/\+----+
                             /       \
                            /         \ 
                           /           \
                          / (C2P)       \
                 +----------------+      \
                 |    AS 4(P4)    |       \
                 ++/\+--+/\+--+/\++        \
                   /     |      \           \
         P2[AS 2] /      |       \           \
                 /       |        \           \
                / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
+----------------+       |          \           \    
|    AS 2(P2)    |       | P1[AS 1]  \           \
+----------+/\+--+       | P6[AS 1]   \           \
             \           | NO_EXPORT   \           \
     P1[AS 1] \          |              \           \
     NO_EXPORT \         |               \           \
                \ (C2P)  | (C2P/P2P)(C2P) \     (C2P) \
              +----------------+        +----------------+
              |  AS 1(P1, P6)  |        |    AS 5(P5)    |
              +----------------+        +----------------+
~~~~~~~~~~
{: #no-export title="Limited propagation of prefixes caused by NO_EXPORT."} 

{{no-export}} presents a scenario where the limited propagation of prefixes occurs due to the NO_EXPORT community attribute. In this scenario, AS 1 is a customer of AS 2, AS 2 is a customer of AS 4, AS 4 is a customer of AS 3, and AS 5 is a customer of both AS 3 and AS 4. The relationship between AS 1 and AS 4 can be either customer-to-provider (C2P) or peer-to-peer (P2P). AS 1 advertises prefixes P1 to AS 2 and adds the NO_EXPORT community attribute to the BGP advertisement sent to AS 2, preventing AS 2 from further propagating the route for prefix P1 to AS 4. Similarly, AS 1 adds the NO_EXPORT community attribute to the BGP advertisement sent to AS 4, resulting in AS 4 not propagating the route for prefix P6 to AS 3. Consequently, AS 4 only learns the route for prefix P1 from AS 1 in this scenario. Suppose AS 1 and AS 4 have deployed inter-domain SAV while other ASes have not, and AS 4 has deployed EFP-uRPF at its customer interfaces. 

In this scenario, existing uRPF-based SAV mechanisms would block the traffic with P1 as source addresses improperly, and thus suffer from the problem of false positives {{inter-domain-ps}}. If the inter-domain SAVNET architecture is deployed, AS 1 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P1 as source addresses can arrive at the interfaces facing AS 1 and AS 2. As a result, the false positive problem can be avoided.

### Hidden Prefixes

~~~~~~~~~~
                                +----------------+
                Anycast Server+-+    AS 3(P3)    |
                                +-+/\----+/\+----+
                                   /       \
                         P3[AS 3] /         \ P3[AS 3]
                                 /           \
                                / (C2P)       \
                       +----------------+      \
                       |    AS 4(P4)    |       \
                       ++/\+--+/\+--+/\++        \
          P6[AS 1, AS 2] /     |      \           \
               P2[AS 2] /      |       \           \
                       /       |        \           \
                      / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
      +----------------+       |          \           \    
User+-+    AS 2(P2)    |       | P1[AS 1]  \           \
      +----------+/\+--+       | P6[AS 1]   \           \
          P6[AS 1] \           | NO_EXPORT   \           \
           P1[AS 1] \          |              \           \
           NO_EXPORT \         |               \           \
                      \ (C2P)  | (C2P)    (C2P) \     (C2P) \
                    +----------------+        +----------------+
       Edge Server+-+  AS 1(P1, P6)  |        |    AS 5(P5)    |
                    +----------------+        +----------------+
P3 is the anycast prefix and is only advertised by AS 3 through BGP.
~~~~~~~~~~
{: #dsr title="A Direct Server Return (DSR) scenario."}

{{dsr}} illustrates a direct server return (DSR) scenario where the anycast IP prefix P3 is only advertised by AS 3 through BGP. In this example, AS 3 is the provider of AS 4 and AS 5, AS 4 is the provider of AS 1, AS 2, and AS 5, and AS 2 is the provider of AS 1. AS 1 and AS 4 have deployed inter-domain SAV, while other ASes have not. When users in AS 2 send requests to the anycast destination IP, the forwarding path is AS 2->AS 4->AS 3. The anycast servers in AS 3 receive the requests and tunnel them to the edge servers in AS 1. Finally, the edge servers send the content to the users with source addresses in prefix P3. The reverse forwarding path is AS 1->AS 4->AS 2. 

In this scenario, existing uRPF-based mechanisms will improperly block the legitimate response packets from AS 1 at the customer interface of AS 4 facing AS 1 {{inter-domain-ps}}. In contrast, if the inter-domain SAVNET architecture is deployed, AS 1 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P3 as source addresses can arrive at the interfaces facing AS 1 and AS 3. As a result, the legitimate response packets with P3 as source addresses from AS 1 can be allowed and the false positive problem can be avoided.

### Reflection Attacks {#reflection_attack_customer}

~~~~~~~~~~
                                        +----------------+
                                        |    AS 3(P3)    |
                                        +-+/\----+/\+----+
                                           /       \
                                          /         \ 
                                         /           \
                                        / (C2P)       \
                               +----------------+      \
                               |    AS 4(P4)    |       \
                               ++/\+--+/\+--+/\++        \
                  P6[AS 1, AS 2] /     |      \           \
                       P2[AS 2] /      |       \           \
                               /       |        \           \
                              / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
              +----------------+       |          \           \    
Attacker(P1')-+    AS 2(P2)    |       | P1[AS 1]  \           \
              +----------+/\+--+       | P6[AS 1]   \           \
                  P6[AS 1] \           | NO_EXPORT   \           \
                   P1[AS 1] \          |              \           \
                   NO_EXPORT \         |               \           \
                              \ (C2P)  | (C2P)    (C2P) \     (C2P) \ 
                          +----------------+          +----------------+
                  Victim+-+  AS 1(P1, P6)  |  Server+-+    AS 5(P5)    |
                          +----------------+          +----------------+
P1' is the spoofed source prefix P1 by the attacker which is inside of 
AS 2 or connected to AS 2 through other ASes.
~~~~~~~~~~
{: #customer-reflection-attack title="A scenario of reflection attacks by source address spoofing within a customer cone."}

{{customer-reflection-attack}} depicts the scenario of reflection attacks by source address spoofing within a customer cone. The reflection attack by source address spoofing takes place within AS 4's customer cone, where the attacker spoofs the victim's IP address (P1) and sends requests to servers' IP address (P5) that are designed to respond to such requests. As a result, the server sends overwhelming responses back to the victim, thereby exhausting its network resources. The arrows in {{customer-reflection-attack}} illustrate the commercial relationships between ASes. AS 3 serves as the provider for AS 4 and AS 5, while AS 4 acts as the provider for AS 1, AS 2, and AS 5. Additionally, AS 2 is the provider for AS 1. Suppose AS 1 and AS 4 have deployed inter-domain SAV, while the other ASes have not.

In this scenario, EFP-uRPF with algorithm A/B will improperly permit the spoofing attacks originating from AS 2 {{inter-domain-ps}}. If the inter-domain SAVNET architecture is deployed, AS 1 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P1 as source addresses can only arrive at the interface facing AS 1. Therefore, at the interface of AS 4 facing AS 2, the spoofing traffic can be blocked.

### Direct Attacks {#direct_attack_customer}

~~~~~~~~~~
                                        +----------------+
                                        |    AS 3(P3)    |
                                        +-+/\----+/\+----+
                                           /       \
                                          /         \ 
                                         /           \
                                        / (C2P)       \
                               +----------------+      \
                               |    AS 4(P4)    |       \
                               ++/\+--+/\+--+/\++        \
                  P6[AS 1, AS 2] /     |      \           \
                       P2[AS 2] /      |       \           \
                               /       |        \           \
                              / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
              +----------------+       |          \           \    
Attacker(P5')-+    AS 2(P2)    |       | P1[AS 1]  \           \
              +----------+/\+--+       | P6[AS 1]   \           \
                  P6[AS 1] \           | NO_EXPORT   \           \
                   P1[AS 1] \          |              \           \
                   NO_EXPORT \         |               \           \
                              \ (C2P)  | (C2P)    (C2P) \     (C2P) \
                            +----------------+        +----------------+
                    Victim+-+  AS 1(P1, P6)  |        |    AS 5(P5)    |
                            +----------------+        +----------------+
P5' is the spoofed source prefix P5 by the attacker which is inside of 
AS 2 or connected to AS 2 through other ASes.
~~~~~~~~~~
{: #customer-direct-attack title="A scenario of the direct attacks by source address spoofing within a customer cone."}

{{customer-direct-attack}} portrays a scenario of direct attacks by source address spoofing within a customer cone and is used to analyze the gaps of uRPF-based mechanisms below. The direct attack by source address spoofing takes place within AS 4's customer cone, where the attacker spoofs a source address (P5) and directly targets the victim's IP address (P1), overwhelming its network resources. The arrows in {{customer-direct-attack}} illustrate the commercial relationships between ASes. AS 3 serves as the provider for AS 4 and AS 5, while AS 4 acts as the provider for AS 1, AS 2, and AS 5. Additionally, AS 2 is the provider for AS 1. Suppose AS 1 and AS 4 have deployed inter-domain SAV, while the other ASes have not.

In this scenario, EFP-uRPF with algorithm A/B will improperly permit the spoofing attacks {{inter-domain-ps}}. If the inter-domain SAVNET architecture is deployed, AS 5 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P5 as source addresses can arrive at the interface facing AS 3 and AS 5. Therefore, at the interface of AS 4 facing AS 2, the spoofing traffic can be blocked.

## SAV at Provider/Peer Interfaces {#sav_at_p}

In order to prevent packets with spoofed source addresses from the provider/peer AS, ACL-based ingress filtering, Loose uRPF, and/or source-based RTBH filtering can be deployed {{nist}}. {{inter-domain-ps}} exposes the limitations of ACL-based ingress filtering, source-based RTBH filtering, and Loose uRPF for SAV at provider/peer interfaces in scenarios of source address spoofing attacks from provider/peer AS. The source address spoofing attacks from provider/peer AS include reflection attacks from provider/peer AS and direct attacks from provider/peer AS. The following showcases that the inter-domain SAVNET architecture can avoid false negatives in these scenarios.

### Reflection Attacks {#reflect_attack_p}

~~~~~~~~~~
                                  +----------------+
                   Attacker(P1')+-+    AS 3(P3)    |
                                  +-+/\----+/\+----+
                                     /       \
                                    /         \ 
                                   /           \
                                  / (C2P/P2P)   \
                         +----------------+      \
                         |    AS 4(P4)    |       \
                         ++/\+--+/\+--+/\++        \
            P6[AS 1, AS 2] /     |      \           \
                 P2[AS 2] /      |       \           \
                         /       |        \           \
                        / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
        +----------------+       |          \           \    
Server+-+    AS 2(P2)    |       | P1[AS 1]  \           \
        +----------+/\+--+       | P6[AS 1]   \           \
            P6[AS 1] \           | NO_EXPORT   \           \
             P1[AS 1] \          |              \           \
             NO_EXPORT \         |               \           \
                        \ (C2P)  | (C2P)    (C2P) \     (C2P) \
                      +----------------+        +----------------+
              Victim+-+  AS 1(P1, P6)  |        |    AS 5(P5)    |
                      +----------------+        +----------------+
P1' is the spoofed source prefix P1 by the attacker which is inside of 
AS 3 or connected to AS 3 through other ASes.
~~~~~~~~~~
{: #reflection-attack-p title="A scenario of reflection attacks by source address spoofing from provider/peer AS."}

{{reflection-attack-p}} depicts the scenario of reflection attacks by source address spoofing from provider/peer AS. In this case, the attacker spoofs the victim's IP address (P1) and sends requests to servers' IP address (P2) that respond to such requests. The servers then send overwhelming responses back to the victim, exhausting its network resources. The arrows in {{reflection-attack-p}} represent the commercial relationships between ASes. AS 3 acts as the provider or lateral peer of AS 4 and the provider for AS 5, while AS 4 serves as the provider for AS 1, AS 2, and AS 5. Additionally, AS 2 is the provider for AS 1. Suppose AS 1 and AS 4 have deployed inter-domain SAV, while the other ASes have not.

Both ACL-based ingress filtering and source-based RTBH filtering will induce additional operational overhead, and Loose uRPF may improperly permit spoofed packets {{inter-domain-ps}}. If the inter-domain SAVNET architecture is deployed, AS 1 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P1 as source addresses can arrive at the interface facing AS 1 and AS 2. Therefore, at the interface of AS 4 facing AS 3, the spoofing traffic can be blocked.

### Direct Attacks {#direct_attack_p}

~~~~~~~~~~
                          +----------------+
           Attacker(P2')+-+    AS 3(P3)    |
                          +-+/\----+/\+----+
                             /       \
                            /         \ 
                           /           \
                          / (C2P/P2P)   \
                 +----------------+      \
                 |    AS 4(P4)    |       \
                 ++/\+--+/\+--+/\++        \
    P6[AS 1, AS 2] /     |      \           \
         P2[AS 2] /      |       \           \
                 /       |        \           \
                / (C2P)  |         \ P5[AS 5]  \ P5[AS 5]
+----------------+       |          \           \    
|    AS 2(P2)    |       | P1[AS 1]  \           \
+----------+/\+--+       | P6[AS 1]   \           \
    P6[AS 1] \           | NO_EXPORT   \           \
     P1[AS 1] \          |              \           \
     NO_EXPORT \         |               \           \
                \ (C2P)  | (C2P)    (C2P) \     (C2P) \
              +----------------+        +----------------+
      Victim+-+  AS 1(P1, P6)  |        |    AS 5(P5)    |
              +----------------+        +----------------+
P2' is the spoofed source prefix P2 by the attacker which is inside of 
AS 3 or connected to AS 3 through other ASes.
~~~~~~~~~~
{: #direct-attack-p title="A scenario of direct attacks by source address spoofing from provider/peer AS."}

{{direct-attack-p}} showcases a scenario of direct attack by source address spoofing from provider/peer AS. In this case, the attacker spoofs another source address (P2) and directly targets the victim's IP address (P1), overwhelming its network resources. The arrows in {{direct-attack-p}} represent the commercial relationships between ASes. AS 3 acts as the provider or lateral peer of AS 4 and the provider for AS 5, while AS 4 serves as the provider for AS 1, AS 2, and AS 5. Additionally, AS 2 is the provider for AS 1. Suppose AS 1 and AS 4 have deployed inter-domain SAV, while the other ASes have not.

Also, in this scenario, Both ACL-based ingress filtering and source-based RTBH filtering will induce additional operational overhead, and Loose uRPF may improperly permit spoofed packets {{inter-domain-ps}}. If the inter-domain SAVNET architecture is deployed, AS 2 can communicate the SAV-specific information to AS 4 and AS 4 will be aware that the traffic with P2 as source addresses can only arrive at the interface facing AS 2. Therefore, at the interface of AS 4 facing AS 3, the spoofing traffic can be blocked.

# Partial/Incremental Deployment Considerations

The inter-domain SAVNET architecture MUST ensure support for partial/incremental deployment as it is not feasible to deploy it simultaneously in all ASes. The partial/incremental deployment of the inter-domain SAVNET architecture consists of different aspects, which include the partial/incremental deployment of the architecture and the partial/incremental deployment of the information sources. 

Within the architecture, the general information like the prefixes and topological information from RPKI ROA Objects and ASPA Objects and the routing information from the RIB can be obtained locally when the corresponding sources are available. Even when both SAV-specific Information and the information from RPKI ROA Objects and ASPA Objects are not available, the routing information from the RIB can be used to generate SAV rules.

Furthermore, it is not mandatory for all ASes to deploy SAV Agents for SAV-specific Information. Instead, a SAV Agent should be able to effortlessly establish a logical neighboring relationship with another AS that has deployed a SAV Agent. The connections for communicating SAV-specific Information can be achieved by manual configurations set by operators or an automatic neighbor discovery mechanism. This flexibility enables the architecture to accommodate varying degrees of deployment, promoting interoperability and collaboration among participating ASes. During the partial/incremental deployment of SAV Agent, the SAV-specific Information for the ASes which do not deploy SAV Agent can not be obtained. To protect the prefixes of these ASes, inter-domain SAVNET architecture can use the SAV-related information from the general information in the SIB to generate SAV rules. At least, the routing information from the RIB can be always available in the SIB.

As more ASes adopt the inter-domain SAVNET architecture, the "deployed area" expands, thereby increasing the collective defense capability against source address spoofing. Furthermore, if multiple "deployed areas" can be logically interconnected across "non-deployed areas", these interconnected "deployed areas" can form a logical alliance, providing enhanced protection against address spoofing. Especially, along with more ASes deploy SAV Agent and support the communication of SAV-specific Information, the generated SAV rules of the inter-domain SAVNET architecture to protect these ASes will become more accurate, as well as enhancing the protection capability against source address spoofing for the inter-domain SAVNET architecture.

In addition, releasing the SAV functions of the inter-domain SAVNET architecture incrementally is one potential way to reduce the deployment risks and can be considered in its deployment by network operators:

* First, the inter-domain SAVNET can only do the measurement in the data plane and do not take any other actions. Based on the measurement data, the operators can evaluate the effect of the inter-domain SAVNET on the legitimate traffic, including validation accuracy and forwarding performance, as well as the operational overhead.

* Second, the inter-domain SAVNET can open the function to limit the rate of the traffic that is justified as spoofing traffic. The operators can further evaluate the effect of the inter-domain SAVNET on the legitimate traffic and spoofing traffic, such as limiting the rate of all the spoofing traffic without hurting the legitimate traffic.

* Third, when the validation accuracy, forwarding performance, and operational overhead have been verified on a large scale by the live network, the inter-domain SAVNET can open the function to directly block the spoofing traffic that is justified by the SAV table in the data plane.

# Convergence Considerations

Convergence issues SHOULD be carefully considered in inter-domain SAV mechanisms due to the dynamic nature of the Internet. Internet routes undergo continuous changes, and SAV rules MUST proactively adapt to these changes, such as prefix and topology changes, in order to prevent false positives and reduce false negatives. To effectively track these changes, the SIM should promptly collect SAV-related information from various SAV information sources and consolidate them in a timely manner.

In particular, it is essential for the SAV Agents to proactively communicate the changes of the SAV-specific Information between ASes and adapt to route changes promptly. However, during the routing convergence process, the traffic paths of the source prefixes can undergo rapid changes within a short period. The changes of the SAV-specific Information may not be communicated in time between ASes to update SAV rules, false positives or false negatives may happen. Such inaccurate validation is caused by the delays in communicating SAV-specific Information between ASes, which occur due to the factors like packet losses, unpredictable network latencies, or message processing latencies. The design of the SAV-specific communication mechanism should consider these issues to reduce the inaccurate validation.

Besides, for the inter-domain SAVNET architecture, the potential ways to deal with the inaccurate validation issues during the convergence of the SAV-specific communication mechanism is to consider using the information from RPKI ROA Objects and ASPA objects to generate SAV rules until the convergence process of the SAV-specific communication mechanism is finished, since these information is more stable and can help avoid false positives, and thus avoiding the impact to the legitimate traffic.

# Manageability Considerations

It is crucial to consider the operations and management aspects of SAV information sources, the SAV-specific communication mechanism, SIB, SIM, and SAV table in the inter-domain SAVNET architecture. The following guidelines should be followed for their effective management:

First, management interoperability should be supported across devices from different vendors or different releases of the same product, based on a unified data model such as YANG {{RFC6020}}. This is essential because the Internet comprises devices from various vendors and different product releases that coexist simultaneously.

Second, scalable operation and management methods such as NETCONF {{RFC6241}} and syslog protocol {{RFC5424}} should be supported. This is important as an AS may have hundreds or thousands of border routers that require efficient operation and management.

Third, management operations, including default initial configuration, alarm and exception reporting, logging, performance monitoring and reporting for the control plane and data plane, as well as debugging, should be designed and implemented in the protocols or protocol extensions. These operations can be performed either locally or remotely, based on the operational requirements.

By adhering to these rules, the management of SAV information sources and related components can be effectively carried out, ensuring interoperability, scalability, and efficient operations and management of the inter-domain SAVNET architecture.

# Security Considerations {#Security}

In the inter-domain SAVNET architecture, the SAV Agent plays a crucial role in generating and disseminating SAV-specific Messages across different ASes. To safeguard against the potential risks posed by a malicious AS generating incorrect or forged SAV-specific Messages, it is important for the SAV Agents to employ security authentication measures for each received SAV-specific Message. The majour security threats faced by inter-domain SAVNET can be categorized into two aspects: session security and content security. Session security pertains to verifying the identities of both parties involved in a session and ensuring the integrity of the session content. Content security, on the other hand, focuses on verifying the authenticity and reliability of the session content, thereby enabling the identification of forged SAV-specific Messages.

The threats to session security include:

* Session identity impersonation: This occurs when a malicious router deceitfully poses as a legitimate peer router to establish a session with the targeted router. By impersonating another router, the malicious entity can gain unauthorized access and potentially manipulate or disrupt the communication between the legitimate routers.

* Session integrity destruction: In this scenario, a malicious intermediate router situated between two peering routers intentionally tampers with or destroys the content of the relayed SAV-specific Message. By interfering with the integrity of the session content, the attacker can disrupt the reliable transmission of information, potentially leading to miscommunication or inaccurate SAV-related data being propagated.

The threats to content security include:

* Message alteration: A malicious router has the ability to manipulate or forge any portion of a SAV-specific Message. For example, the attacker may employ techniques such as using a spoofed Autonomous System Number (ASN) or modifying the AS Path information within the message. By tampering with the content, the attacker can potentially introduce inaccuracies or deceive the receiving ASes, compromising the integrity and reliability of the SAV-related information.

* Message injection: A malicious router injects a seemingly "legitimate" SAV-specific Message into the communication stream and directs it to the corresponding next-hop AS. This type of attack can be likened to a replay attack, where the attacker attempts to retransmit previously captured or fabricated messages to manipulate the behavior or decisions of the receiving ASes. The injected message may contain malicious instructions or false information, leading to incorrect SAV rule generation or improper validation.

* Path deviation: A malicious router intentionally diverts a SAV-specific Message to an incorrect next-hop AS, contrary to the expected path defined by the AS Path. By deviating from the intended routing path, the attacker can disrupt the proper dissemination of SAV-related information and introduce inconsistencies or conflicts in the validation process. This can undermine the effectiveness and accuracy of source address validation within the inter-domain SAVNET architecture.

Overall, inter-domain SAVNET shares similar security threats with BGP and can leverage existing BGP security mechanisms to enhance both session and content security. Session security can be enhanced by employing session authentication mechanisms used in BGP, such as MD5, TCP-AO, or Keychain. Similarly, content security can benefit from the deployment of existing BGP security mechanisms like RPKI, BGPsec, and ASPA. While these mechanisms can address content security threats, their widespread deployment is crucial. Until then, it is necessary to develop an independent security mechanism specifically designed for inter-domain SAVNET. One potential approach is for each origin AS to calculate a digital signature for each AS path and include these digital signatures within the SAV-specific Messages. Upon receiving a SAV-specific Message, the SAV Agent can verify the digital signature to ascertain the message's authenticity. Furthermore, it is worth noting that the information channel of the inter-domain SAVNET architecture may need to operate over a network link that is currently under a source address spoofing attack. As a result, it may experience severe packet loss and high latency due to the ongoing attack, and the implementation of the information channel should ensure uninterrupted communication. Detailed security designs and considerations will be addressed in a separate draft, ensuring the robust security of inter-domain SAVNET.

# Privacy Considerations

TBD

# IANA Considerations {#IANA}

This document has no IANA requirements.


# Scope

In this architecture, the choice of protocols used for communication between the SIM and different SAV information sources is not limited. The inter-domain SAVNET architecture presents considerations on how to consolidate SAV-related information from various sources to generate SAV rules and perform SAV using the SAV table in the dataplane. The detailed design and implementation for SAV rule generation and SAV execution depend on the specific inter-domain SAV mechanisms employed.

This document does not cover administrative or business agreements that may be established between the involved inter-domain SAVNET parties. These considerations are beyond the scope of this document. However, it is assumed that authentication and authorization mechanisms can be implemented to ensure that only authorized ASes can communicate SAV-related information.

# Assumptions

This document makes the following assumptions:

* All ASes where the inter-domain SAVNET is deployed are assumed to provide the necessary connectivity between SAV Agent and any intermediate network elements. However, the architecture does not impose any specific limitations on the form or nature of this connectivity. 

* Congestion and resource exhaustion can occur at various points in the inter-domain networks. Hence, in general, network conditions should be assumed to be hostile. The inter-domain SAVNET architecture must be capable of functioning reliably under all circumstances, including scenarios where the paths for delivering SAV-related information are severely impaired. It is crucial to design the inter-domain SAVNET system with a high level of resilience, particularly under extremely hostile network conditions. The architecture should ensure uninterrupted communication between inter-domain SAV Agents, even when data-plane traffic saturates the link.

* The inter-domain SAVNET architecture does not impose rigid requirements for the SAV information sources that can be used to generate SAV rules. Similarly, it does not dictate strict rules on how to utilize the SAV-related information from diverse sources or perform SAV in the dataplane. Network operators have the flexibility to choose their approaches to generate SAV rules and perform SAV based on their specific requirements and preferences. Operators can either follow the recommendations outlined in the inter-domain SAVNET architecture or manually specify the rules for governing the use of SAV-related information, the generation of SAV rules, and the execution of SAV in the dataplane. 

* The inter-domain SAVNET architecture does not impose restrictions on the selection of the local AS with which AS to communicate SAV-specific Information. The ASes have the flexibility to establish connections for SAV-specific communication based on the manual configurations set by operators or other automatic mechanisms. 

* The inter-domain SAVNET architecture provides the flexibility to accommodate Quality-of-Service (QoS) policy agreements between SAVNET-enabled ASes or local QoS prioritization measures, but it does not make assumptions about their presence. These agreements or prioritization efforts are aimed at ensuring the reliable delivery of SAV-specific Information between SAV Agents. It is important to note that QoS is considered as an operational consideration rather than a functional component of the inter-domain SAVNET architecture. 

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

Many thanks to Alvaro Retana, Kotikalapudi Sriram, Rüdiger Volk, Xueyan Song, Ben Maddison, Jared Mauch, Joel Halpern, Aijun Wang, Jeffrey Haas, Xiangqing Chang, Changwang Lin, Mingxing Liu, Zhen Tan, Yuanyuan Zhang, Yangyang Wang, etc. for their valuable comments on this document.