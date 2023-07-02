---
stand_alone: true
ipr: trust200902
cat: std # Check
submissiontype: IETF
area: General [REPLACE]
wg: Internet Engineering Task Force

docname: draft-wu-savnet-inter-domain-architecture-03

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
  sav-table:
    target: https://datatracker.ietf.org/doc/draft-huang-savnet-sav-table/
    title: Source Address Validation Table Abstraction and Application
    date: 2023
  inter-domain-ps:
    target: https://datatracker.ietf.org/doc/draft-wu-savnet-inter-domain-problem-statement/
    title: Source Address Validation in Inter-domain Networks Gap Analysis, Problem Statement, and Requirements 
    date: 2023
  RFC5635:
  RFC8955:

--- abstract

This document presents an inter-domain SAVNET architecture that serves as a comprehensive framework for the development of inter-domain source address validation (SAV) mechanisms. The proposed architecture empowers AS to generate SAV rules by leveraging SAV-related information from various sources. It integrates existing SAV mechanisms and offers ample design space for new inter-domain SAV mechanisms. Instead of delving into protocol extensions or implementations, this document primarily focuses on defining the architectural components, the interconnections between architectural components, and the SAV information sources and corresponding SAV-related information for deploying an inter-domain SAV mechanism.

--- middle

# Introduction

Attacks based on source IP address spoofing, such as reflective DDoS and flooding attacks, continue to present significant challenges to Internet security. Mitigating these attacks in inter-domain networks requires effective source address validation (SAV). While BCP84 {{RFC3704}} {{RFC8704}} offers some SAV solutions, such as ACL-based ingress filtering and uRPF-based mechanisms, existing inter-domain SAV mechanisms have limitations in terms of validation accuracy and operational overhead in different scenarios {{inter-domain-ps}}.

To address these issues, the inter-domain SAVNET architecture focuses on providing a comprehensive framework and guidelines for the design and implementation of inter-domain SAV mechanisms. By defining the architectural components, relationships, and SAV-related information used in interdomain SAV deployments, this document aims to promote consistency, interoperability, and collaboration among ASes. The inter-domain SAVNET architecture empowers ASes to generate precise SAV rules by leveraging SAV-related information from various sources, including Manual Configurations, Routing Information (such as RPKI ROA Objects and ASPA Objects, and RIB), and SAV-tailored Messages from other ASes. This information consists of legitimate prefixes of ASes and their corresponding legitimate incoming interfaces. By consolidating these information, the inter-domain SAVNET architecture improves the accuracy of SAV rules beyond what can be achieved solely based on the local RIB. Additionally, it ensures that the source addresses of ASes communicating SAV-tailored information with SAV-tailored Messages are also protected.

This document primarily describes a high-level architecture for consolidating SAV-related information from various sources and deploying an inter-domain SAV mechanism between ASes. The document does not specify protocol extensions or implementations. Its purpose is to provide a conceptual framework and guidance for the development of inter-domain SAV mechanisms, allowing implementers to adapt and implement the architecture based on their specific requirements and network environments.

## Scope

In this architecture, the choice of protocols used for communication between the SAVNET agent and different SAV information sources is not limited. The inter-domain SAVNET architecture presents considerations on how to consolidate SAV-related information from various sources to generate SAV rules and perform SAV using the SAV table in the dataplane. The detailed design and implementation for SAV rule generation and SAV execution depend on the specific inter-domain SAV mechanisms employed.

This document does not cover administrative or business agreements that may be established between the involved inter-domain SAVNET parties. These considerations are beyond the scope of this document. However, it is assumed that authentication and authorization mechanisms can be implemented to ensure that only authorized ASes can communicate SAV-related information.

A detailed set of requirements for inter-domain SAV are discussed in {{inter-domain-ps}}. The inter-domain SAVNET architecture is designed to adhere to those requirements, and this document primarily introduces a new scalability requirement in addition to the existing ones.

## Assumptions

This document makes the following assumptions:

* All ASes where the inter-domain SAVNET is deployed are assumed to provide the necessary connectivity between SAVNET agents and any intermediate network elements. However, the architecture does not impose any specific limitations on the form or nature of this connectivity. 

* Congestion and resource exhaustion can occur at various points in the inter-domain networks. Hence, in general, network conditions should be assumed to be hostile. The inter-domain SAVNET architecture must be capable of functioning reliably under all circumstances, including scenarios where the paths for delivering SAV-related information are severely impaired. It is crucial to design the inter-domain SAVNET system with a high level of resilience, particularly under extremely hostile network conditions. The architecture should ensure uninterrupted communication between inter-domain SAVNET agents, even when data-plane traffic saturates the link.

* The inter-domain SAVNET architecture does not impose rigid requirements for the SAV information sources that can be used to generate SAV rules. Similarly, it does not dictate strict rules on how to utilize the SAV-related information from diverse sources or perform SAV in the dataplane. Network operators have the flexibility to choose their approaches to generate SAV rules and perform SAV based on their specific requirements and preferences. Operators can either follow the recommendations outlined in the inter-domain SAVNET architecture or manually specify the rules for governing the use of SAV-related information, the generation of SAV rules, and the execution of SAV in the dataplane. 

* The inter-domain SAVNET architecture does not impose restrictions on the selection of the local AS with which AS to communicate SAV-tailored information. The ASes have the flexibility to establish SAV-tailored protocol connections based on the manual configurations set by operators or other automatic mechanisms. 

* The inter-domain SAVNET architecture provides the flexibility to accommodate Quality-of-Service (QoS) policy agreements between SAVNET-enabled ASes or local QoS prioritization measures, but it does not make assumptions about their presence. These agreements or prioritization efforts are aimed at ensuring the reliable delivery of inter-domain SAVNET messages between SAVNET agents. It is important to note that QoS is considered as an operational consideration rather than a functional component of the inter-domain SAVNET architecture. 

* The information and management channels are loosely coupled and are used for collecting SAV-related information from different sources, and how the inter-domain SAVNET synchronize the management and operation configurations is out of scope of this document.

## Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

{:vspace}
SAV Rule:
: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: 
: The table or data structure that implements the SAV rules and is used for source address validation in the data plane. 

Real Forwarding Paths: 
: The paths that the legitimate traffic goes through in the data plane.

SAV-related information:
: The information is used to generate SAV rules and can be from various sources, such as Manual Configurations, RIB, SAV-tailored Messages of ASes, etc.

SAV-tailored Message: 
: The message is used to carry the SAV-tailored information between ASes and can be communicated following a dedicated SAV communication protocol.

SAVNET Agent:
: Any SAVNET-aware software module capable of collecting SAV-related information. It can be a module to accept Manual Configurations or process routing information, SAV-tailored protocol speaker, or, as a logical agent. 

Improper block: 
: The validation results that the packets with legitimate source addresses are blocked improperly due to inaccurate SAV rules.

Improper permit: 
: The validation results that the packets with spoofed source addresses are permitted improperly due to inaccurate SAV rules.

SAV Information Base: 
: A table or data structure for storing SAV-related information collected from various sources.

SAV Information Base Management Protocol: 
: A protocol which is used to operate and manage the SAV Information Base.

# Design Goals

The inter-domain SAVNET architecture aims to improve SAV accuracy and facilitate partial deployment with low operational overhead, as well as supporting developing a unified SAV-tailored protocol to communicate SAV-tailored messages between ASes. The overall goal can be broken down into the following aspects:

* First, the inter-domain SAVNET architecture should learn SAV-related information that consists of real forwarding paths or permissible paths of prefixes that can cover the real forwarding paths, and generate accurate and finer-grained SAV rules automatically based on the learned information to avoid improper block and reduce improper permit as much as possible.

* Second, the inter-domain SAVNET architecture should provide sufficient protection for the source prefixes of ASes that deploy it, even if only a portion of Internet implements the architecture.

* Third, the inter-domain SAVNET architecture should adapt to dynamic networks and asymmetric routing scenarios automatically.

* Fourth, the inter-domain SAVNET architecture should support to communicate SAV-tailored information with a unified SAV-tailored protocol between ASes.

* Last, the inter-domain SAVNET architecture should scale independently from the SAV information sources and the growth of the deployed Internet devices.

SAV-related information can be obtained from different sources, including RIB, Manual Configurations, or SAV-tailored Messages communicated between ASes. The inter-domain SAVNET architecture consolidates SAV-related information from multiple sources and generates SAV rules automatically, and adapts proactively to route changes in a timely manner to achieve the goals outlined above.

Other design goals, such as low operational overhead and easy implementation, are also very important and should be considered in specific protocols or protocol extensions and are out of scope for this document.

# Inter-domain SAVNET Architecture 

~~~~~~~~~~
                                         +--------------------+
                                         |     Other ASes     |
                                         +--------------------+
                                                   |SAV-tailored
                                                   |Messages
+-------------------------------------------------------------+                                       
|    |YANG |CLI |SMP    |RIB |ROA |ASPA            |       AS |
|   \/    \/   \/      \/   \/   \/               \/          |
| +---------------+ +-----------------+ +-------------------+ |
| |     Manual    | |     Routing     | |   SAV-tailored    | |
| | Configuration | |   Information   | |   Information     | |
| +---------------+ +-----------------+ +-------------------+ |
|         |                   |                    |          |
|        \/                  \/                   \/          |
| +---------------------------------------------------------+ |
| | +-----------------------------------------------------+ | |
| | |                  SAV Information Base               | | |
| | +-----------------------------------------------------+ | |
| |                SAV Information Base Manager             | |
| +---------------------------------------------------------+ |
|                              |SAV Rules                     |
|                             \/                              |
|   +-----------------------------------------------------+   |
|   |                      SAV Table                      |   |
|   +-----------------------------------------------------+   |
+-------------------------------------------------------------+
~~~~~~~~~~
{: #arch title="The inter-domain SAVNET architecture"}

{{arch}} shows the overview of the inter-domain SAVNET architecture. The inter-domain SAVNET architecture collects SAV-related information from multiple sources, which can be categorized into three types: Manual Configuration, Routing Information, and SAV-tailored Information. The SAV information base manager (SIM) uses the collected SAV-related information to maintain the SAV Information Base (SIB), and then generate SAV rules based on the SIB and 
fill out the SAV table in the dataplane.

Manual Configuration can be conducted by network operators using various methods such as YANG, Command-Line Interface (CLI), and SIB Management Protocol (SMP) like remote triggered black hole (RTBH) {{RFC5635}} and FlowSpec {{RFC8955}}. The Routing Information can be obtained within an AS and may come from RIB, Routing Information Messages, or RPKI ROA objects and ASPA objects. And SAV-tailored Information, which contains real forwarding paths of the prefixes, is transmitted using SAV-tailored Messages from other ASes. 

It is important to note that the inter-domain SAVNET architecture does not require all the information sources to generate SAV rules. All of the sources, including SAV-tailored Messages, are optional depending on their availability and the operational needs of the inter-domain SAV mechanisms. However, the SAV Information Base Manager (SIM) and SAV table are necessary components for generating SAV rules and performing SAV.

Additionally, inter-domain SAVNET architecture does not prescribe any specific deployment models.

## SAV Information Base {#sib-sec}

The SAV Information Base (SIB) is managed by the SAV Information Base Manager, which consolidates SAV-related information from various sources. Each row of the SIB contains an index, prefix, AS-level valid incoming interface for the prefix, incoming direction, and the corresponding sources of these SAV-related information. The incoming direction consists of customer, provider, and peer.

~~~~~~~~~~
+-----+------+------------------+---------+---------------------------+
|Index|Prefix|AS-level Interface|Direction|   SAV Information Source  |
+-----+------+------------------+---------+---------------------------+
|  0  |  P3  |      Itf.1       |Provider |            RIB            | 
+-----+------+------------------+---------+---------------------------+
|  1  |  P2  |      Itf.2       |Customer |            RIB            |
+-----+------+------------------+---------+---------------------------+
|  2  |  P1  |      Itf.2       |Customer |    SAV-tailored Message   |
+-----+------+------------------+---------+---------------------------+
|  3  |  P1  |      Itf.3       |Customer | Manual Configuration, RIB |
+-----+------+------------------+---------+---------------------------+
|  4  |  P6  |      Itf.2       |Customer |            RIB            |
+-----+------+------------------+---------+---------------------------+
|  5  |  P6  |      Itf.3       |Customer | SAV-tailored Message, RIB |
+-----+------+------------------+---------+---------------------------+
|  6  |  P5  |      Itf.4       |Customer |RPKI ROA Obj. and ASPA Obj.|
|     |      |                  |         |            RIB            |
+-----+------+------------------+---------+---------------------------+
|  7  |  P5  |      Itf.1       |Provider |            RIB            |
+-----+------+------------------+---------+---------------------------+
~~~~~~~~~~
{: #sib title="An example for the SAV information base in AS 4"}

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

In order to provide a clear illustration of the SIB, {{sib}} depicts an example of an SIB established in AS 4. As shown in {{as-topo}}, AS 4 has four AS-level interfaces, each connected to a different AS. Specifically, Itf.1 is connected to AS 3, Itf.2 to AS 2, Itf.3 to AS 1, and Itf.4 to AS 5. The arrows in the figure represent the commercial relationships between ASes. AS 3 is the provider of AS 4 and AS 5, while AS 4 is the provider of AS 1, AS 2, and AS 5, and AS 2 is the provider of AS 1. Assuming prefixes P1, P2, P3, P4, P5, and P6 are all the prefixes in the network. 

For example, in {{sib}}, the row with index 0 indicates prefix P3's valid incoming interface is Itf.1, the ingress direction of P3 is AS 4's provider AS (AS 3), and these information is from the RIB. Note that the same SAV-related information may have multiple sources and the SIB records them all, such as the rows with indexes 3, 5, and 6.
        
In sum, the SIB can be consolidated according to the SAV-related information from the following sources:

* Manual Configuaration: the SIB can obtain SAV-related configurations from the Manual Configurations of network operators using various approaches, such as YANG {{RFC6020}}, Command-Line Interface (CLI), and the SIM like RTBH {{RFC5635}} and FlowSpec {{RFC8955}}.

* SAV-tailored Message: the SIB can obtain the real forwarding paths of prefixes from the processed SAV-tailored Messages from other ASes, which are communicated between ASes with the SAV-tailored protocol.

* RPKI ROA Objects and ASPA Objects: the SIB can obtain the topological information from the RPKI ROA objects and ASPA objects over the local routing instance with the local cache of RPKI objects or RPKI validation process.

* Routing Information Base (RIB): the SIB can obtain the prefixes and their permissible routes including the optimal ones and optional ones from the RIB.

~~~~~~~~~~
+---------------------------------------+----------+
|       SAV Information Sources         |Priorities|
+---------------------------------------+----------+
|         Manual Configuration          |     1    | 
+---------------------------------------+----------+
|         SAV-tailored Message          |     2    |
+---------------------------------------+----------+
|   RPKI ROA Objects and ASPA Objects   |     3    |
+---------------------------------------+----------+
|                  RIB                  |     4    |
+---------------------------------------+----------+
~~~~~~~~~~
{: #sav_src title="Priority ranking for the SAV information sources"}

The SIB is maintained according to the SAV-related information from all the above information sources or part of them. It explicitly records the sources of the SAV-related information and can be compressed to reduce its size. Inter-domain SAVNET architecture allows operators to specify how to use the SAV-related information in the SIB by manual configurations.

Notably, the sources of SAV information are not equal in their effectiveness. Finer-grained information sources, such as SAV-tailored Messages, play a crucial role in generating more accurate SAV rules, thereby reducing the risk of improper permits or blocks. In light of this consideration, the inter-domain SAVNET architecture offers a default approach for leveraging information from diverse sources within the SIB. {{sav_src}} presents a priority ranking for the SAV information sources within the SIB.

The inter-domain SAVNET architecture utilizes these priorities to generate SAV rules based on the available SAV information sources. For example, in the case of provider interfaces where loose SAV rules are applicable, inter-domain SAVNET may utilize SAV-related information from all available sources (e.g., rows with index 0 and 7 in {{sib}}) to generate rules. However, for customer interfaces that necessitate stricter SAV rules, the inter-domain SAVNET architecture may prioritize the utilization of information from sources with higher priorities for the same SAV-related information. This prioritization can be seen in the selection of rows with index 1, 3, 5, and 6 in {{sib}}.

For instance, in {{sib}}, the inter-domain SAVNET architecture uses the row with index 1 to generate SAV rules since only SAV-related information from the RIB is available for prefix P2. It is important to note that network operators retain the ability to specify the utilization methods for SAV-related information, including defining the priorities assigned to different SAV information sources.

## SAV-tailored Protocol

~~~~~~~~~~
+-----------------------+                +-----------------------+
|          AS           |  SAV-tailored  |          AS           |
| +------------------+  |    Messages    |  +------------------+ |
| |   SAV-tailored   |<-|----------------|->|   SAV-tailored   | |
| | Protocol Speaker |  |                |  | Protocol Speaker | |
| +------------------+  |                |  +------------------+ |
+-----------------------+                +-----------------------+
~~~~~~~~~~
{: #sav_msg title="Communicating SAV-tailored Messages between SAV-tailored protocol speakers"}

As shown in {{sav_msg}}, SAV-tailored Messages are used to propagate or originate the real forwarding paths of prefixes between SAV-tailored Protocol Speakers. Within an AS, the SAV-tailored Protocol Speaker can obtain the next hop of the corresponding prefixes based on the routing table from the local RIB and use SAV-tailored Messages to carry the next hops of the corresponding prefixes and propagate them between ASes.

To generate the real forwarding paths of prefixes, the SAV-tailored Protocol Speaker connects to other SAV-tailored Protocol Speakers in other ASes, receives, processes, generates, or terminates SAV-tailored Messages. Then, it obtains the ASN, the prefixes, the AS-level interfaces to receive the messages, and their incoming AS direction for maintaining the SIB. It is important to note that the SAV-tailored Protocol Speaker within an AS has the capability to establish connections with multiple SAV-tailored Protocol Speakers from different ASes, relying on either manual configurations by operators or an automatic mechanism.

Moreover, the preferred AS paths of an AS may change over time due to route changes, including source prefix change and AS path change. The SAV-tailored Protocol Speaker should launch SAV-tailored Messages to adapt to the route changes in a timely manner. Inter-domain SAVNET should handle route changes carefully to avoid improper block. The reasons for leading to improper block may include late detection of route changes, delayed message transmission, or packet losses. However, the detailed design for dealing with route changes is outside the scope of this document.

## SAV Information Base Manager

SAV Information Base Manager (SIM) consolidates SAV-related information from multiple sources and generates SAV rules. It uses the SAV-related information to initiate or update the SIB and generates SAV rules to populate the SAV table in the dataplane according to the SIB. The collection methods of SAV-related information depend on the deployment and implementation of the inter-domain SAV mechanisms and are out of scope for this document.

Using the SIB, SIM produces &lt;Prefix, Interface&gt; pairs to populate the SAV table, which represents the prefix and its legitimate incoming interface. It is worth noting that the interfaces in the SIB are logical AS-level interfaces and need to be mapped to the physical interfaces of border routers within the corresponding AS.

~~~~~~~~~~
+------------------------------------+
| Source Prefix | Incoming Interface |
+---------------+--------------------+
|      P1       |          1         |
+---------------+--------------------+
|      P2       |          2         |
+---------------+--------------------+
|      P3       |          3         |
+------------------------------------+
~~~~~~~~~~
{: #sav_tab title="An example of SAV table"}

{{sav_tab}} shows an example of the SAV table. The packets coming from other ASes will be validated by the SAV table. The router looks up each packet's source address in its local SAV table and gets one of three validity states: "Valid", "Invalid" or "Unknown". "Valid" means that there is a source prefix in SAV table covering the source address of the packet and the valid incoming interfaces covering the actual incoming interface of the packet. According to the SAV principle, "Valid" packets will be forwarded. "Invalid" means there is a source prefix in SAV table covering the source address, but the incoming interface of the packet does not match any valid incoming interface so that such packets will be dropped. "Unknown" means there is no source prefix in SAV table covering the source address. The packet with "unknown" addresses can be dropped or permitted, which depends on the choice of operators. The structure and usage of SAV table can refer to {{sav-table}}.

## Management Channel and Information Channel

~~~~~~~~~~
+------+
|      |   Management Channel    +--------------------------------+ 
|      |<========================|        Network Operators       |
|      |                         +--------------------------------+
|      |
|      |   Information Channel   +--------------------------------+
|      |<----------------------->|  SAV-tailored Protocol Speaker |
|SAVNET|                         +--------------------------------+
|Agent |
|      |   Information Channel   +--------------------------------+
|      |<------------------------| RPKI ROA and ASPA Cache Server |
|      |                         +--------------------------------+
|      |
|      |   Information Channel   +--------------------------------+
|      |<------------------------|               RIB              |
|      |                         +--------------------------------+
+------+
~~~~~~~~~~
{: #sav_agent_config title="The management channel and information channel for collecting SAV-related information from different SAV information sources"}

As illustrated in {{sav_agent_config}}, there are different modules to receive SAV-related information from network operators, SAV-tailored Protocol Speaker, RPKI ROA objects and ASPA objects, and RIB: Management Channel and Information Channel. SAVNET Agent is a general term for the modules like manual configuration processor, SAV-tailored Protocol Speaker, RPKI validation process, and module for accessing RIB.

The primary purpose of the management channel is to deliver SAV-related information from Manual Configurations of network operators and SAVNET-specialized configurations. Examples of such information include, but are not limited to: 

* SAV configurations using YANG.

* SAV configurations using CLI.

* SAV configurations using SMP.

* SAVNET operation and management.

* Inter-domain SAVNET provisioning.

Note that the information can be delivered at anytime and is required reliable delivery for the management channel implementation.

The information channel serves as a means to transmit SAV-related information from various sources, including SAV-tailored messages processed by the SAV-tailored protocol speaker, RPKI ROA objects and ASPA objects for RPKI ROA and ASPA Cache Server, and RIB. An SAVNET Agent can establish connections with multiple SAVNET Agents belonging to different ASes based on manual configurations by operators or an automatic mechanism. Additionally, it can carry telemetry information, such as metrics pertaining to forwarding performance, the count of spoofing packets and discarded packets, provided that the SAVNET Agent has access to such data. Moreover, the information channel can include information regarding the prefixes associated with the spoofing traffic, as observed by the SAVNET Agent up until the most recent time. It is worth noting that the information channel may need to operate over a network link that is currently under a source address spoofing attack. As a result, it may experience severe packet loss and high latency due to the ongoing attack.

# Partial Deployment

The inter-domain SAVNET architecture MUST ensure support for partial deployment as it is not feasible to deploy it simultaneously in all ASes. Within the architecture, certain information such as operator configurations, topological information, and routing information can be obtained locally when the corresponding sources are available. Furthermore, it is not mandatory for all ASes to deploy SAV-tailored Protocol Speakers even for SAV-tailored information. Instead, a SAV-tailored Protocol Speaker can effortlessly establish a logical neighboring relationship with another AS that has deployed a SAV-tailored Protocol Speaker. This flexibility enables the architecture to accommodate varying degrees of deployment, promoting interoperability and collaboration among participating ASes.

In general, the ASes that implement the inter-domain SAVNET architecture cannot engage in source address spoofing against each other. Additionally, non-deployed ASes are unable to send spoofed packets to an AS by falsifying its source addresses along the paths with deployed ASes. As more ASes adopt the inter-domain SAVNET architecture, the "deployed area" expands, thereby increasing the collective defense capability against source address spoofing. Furthermore, if multiple "deployed areas" can be logically interconnected across "non-deployed areas", these interconnected "deployed areas" can form a logical alliance, providing enhanced protection against address spoofing.

# Convergence Considerations

Convergence issues SHOULD be carefully considered in inter-domain SAV mechanisms due to the dynamic nature of the Internet. Internet routes undergo continuous changes, and SAV rules MUST proactively adapt to these changes, such as prefix and topology changes, in order to prevent improper block or reduce improper permit actions. To effectively track these changes, the SAVNET Agent should promptly collect SAV-related information from various SAV information sources and the SIM should consolidate them in a timely manner.

When it comes to SAV-related information obtained from Manual Configurations, the SAVNET Agent can directly receive the configurations from network operators and the SIM can promptly generate the corresponding SAV rules.

Regarding Routing Information, the mechanism can rely on the convergence mechanism of routing protocols such as BGP. However, it is important to note that there may be instances of inaccurate validation before the route convergence process is fully completed. This challenge is also observed in existing uRPF-based mechanisms. Although temporary inconsistencies in forwarding paths during the convergence process may occur, it may be acceptable as the real forwarding paths of prefixes will fluctuate during this time.

For SAV-tailored information, it is essential for the SAV-tailored Protocol Speaker to proactively send SAV-tailored Messages and adapt to route changes promptly. However, there are challenges that need to be addressed. First, during the routing convergence process, real forwarding paths can undergo rapid changes within a short period. Relying solely on temporary SAV-tailored information during this time may lead to incorrect block decisions. Additionally, similar to routing protocols, there is a possibility of inaccurate validation during the convergence process of the SAV-tailored protocol due to delays in SAV-tailored Messages. These delays can occur due to factors like packet losses, unpredictable network latencies, or message processing latencies.

To prevent improper block, it is crucial for the inter-domain SAVNET to handle these challenges carefully. One approach is to generate SAV rules automatically based on the available Routing Information until the convergence process of the routing changes and the SAV-tailored protocol is finalized. This ensures that the SAV-tailored protocol adapts to the changing network conditions and avoids making improper blocking decisions during the convergence phase. Furthermore, a dedicated protocol like the SAV-tailored protocol has the capability to control the synchronization timing of SAV-tailored information between ASes. As a result, the SAV-tailored protocol can control the pace of generating new SAV rules and evicting old SAV rules to adoid false positives and reduce false negatives.

# Management Considerations

It is crucial to consider the operations and management aspects of various SAV information sources, the SAV-tailored protocol, SIB, SIM, and SAV table in the inter-domain SAVNET architecture. The following guidelines should be followed for their effective management:

First, management interoperability should be supported across devices from different vendors or different releases of the same product, based on a unified data model such as YANG {{RFC6020}}. This is essential because the Internet comprises devices from various vendors and different product releases that coexist simultaneously.

Second, scalable operation and management methods such as NETCONF {{RFC6241}} and syslog protocol {{RFC5424}} should be supported. This is important as an AS may have hundreds or thousands of border routers that require efficient operation and management.

Third, management operations, including default initial configuration, alarm and exception reporting, logging, performance monitoring and reporting for the control plane and data plane, as well as debugging, should be designed and implemented in the protocols or protocol extensions. These operations can be performed either locally or remotely, based on the operational requirements.

By adhering to these rules, the management of SAV information sources and related components can be effectively carried out, ensuring interoperability, scalability, and efficient operations and management of the inter-domain SAVNET architecture.

# Security Considerations {#Security}

In the inter-domain SAVNET architecture, the SAV-tailored Protocol Speaker plays a crucial role in generating and disseminating SAV-tailored Messages across different ASes. To safeguard against the potential risks posed by a malicious AS generating incorrect or forged SAV-tailored Messages, it is important for the SAV-tailored Protocol Speakers to employ security authentication measures for each received SAV-tailored Message. The security threats faced by inter-domain SAVNET can be categorized into two aspects: session security and content security. Session security pertains to verifying the identities of both parties involved in a session and ensuring the integrity of the session content. Content security, on the other hand, focuses on verifying the authenticity and reliability of the session content, thereby enabling the identification of forged SAV-tailored Messages.

The threats to session security include:

* Session identity impersonation: This occurs when a malicious router deceitfully poses as a legitimate peer router to establish a session with the targeted router. By impersonating another router, the malicious entity can gain unauthorized access and potentially manipulate or disrupt the communication between the legitimate routers.

* Session integrity destruction: In this scenario, a malicious intermediate router situated between two peering routers intentionally tampers with or destroys the content of the relayed SAV-tailored Message. By interfering with the integrity of the session content, the attacker can disrupt the reliable transmission of information, potentially leading to miscommunication or inaccurate SAV-related data being propagated.

The threats to content security include:

* Message alteration: A malicious router has the ability to manipulate or forge any portion of a SAV-tailored Message. For example, the attacker may employ techniques such as using a spoofed Autonomous System Number (ASN) or modifying the AS Path information within the message. By tampering with the content, the attacker can potentially introduce inaccuracies or deceive the receiving ASes, compromising the integrity and reliability of the SAV-related information.

* Message injection: A malicious router injects a seemingly "legitimate" SAV-tailored Message into the communication stream and directs it to the corresponding next-hop AS. This type of attack can be likened to a replay attack, where the attacker attempts to retransmit previously captured or fabricated messages to manipulate the behavior or decisions of the receiving ASes. The injected message may contain malicious instructions or false information, leading to incorrect SAV rule generation or improper validation.

* Path deviation: A malicious router intentionally diverts a SAV-tailored Message to an incorrect next-hop AS, contrary to the expected path defined by the AS Path. By deviating from the intended routing path, the attacker can disrupt the proper dissemination of SAV-related information and introduce inconsistencies or conflicts in the validation process. This can undermine the effectiveness and accuracy of source address validation within the inter-domain SAVNET architecture.

Overall, inter-domain SAVNET shares similar security threats with BGP and can leverage existing BGP security mechanisms to enhance both session and content security. Session security can be enhanced by employing session authentication mechanisms used in BGP, such as MD5, TCP-AO, or Keychain. Similarly, content security can benefit from the deployment of existing BGP security mechanisms like RPKI, BGPsec, and ASPA. While these mechanisms can address content security threats, their widespread deployment is crucial. Until then, it is necessary to develop an independent security mechanism specifically designed for inter-domain SAVNET. One potential approach is for each origin AS to calculate a digital signature for each AS path and include these digital signatures within the SAV-tailored Messages. Upon receiving a SAV-tailored Message, the SAV-tailored Protocol Speaker can verify the digital signature to ascertain the message's authenticity. Detailed security designs and considerations will be addressed in a separate draft, ensuring the robust security of inter-domain SAVNET.

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

Many thanks to Alvaro Retana, Kotikalapudi Sriram, Rüdiger Volk, Xueyan Song, Ben Maddison, Jared Mauch, Joel Halpern, Aijun Wang, Jeffrey Haas, Fang Gao, Mingxing Liu, Zhen Tan, etc. for their valuable comments on this document.