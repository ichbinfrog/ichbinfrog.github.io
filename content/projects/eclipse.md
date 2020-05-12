---
title: "Eclipse/steady — Cloud integration"
date: 2019-10-30T11:55:00+00:00
draft: false
toc: true
images:
tags:
  - overview
---

Over the past decade, the inclusion of open-source libraries has become a staple of the software industry. The RedHat report on the state of enterprise open source[7]  reveals that out of 950 IT leaders worldwide, around 68% of companies are increasing their use of open source components, adopting it in all its domain of activities (from big data to security). However the report also shares the fact that one of the major barriers to adoption of open source comes from the security concerns related to said libraries. At the same time, this enormous growth is also accompanied by the amount of disclosed vulnerabilities [8]. The potential gains of reusing community contributed package can be overshadowed by the cost of assessing and mitigating said weaknesses. 

Project Vulas (recently adopted by the Eclipse foundation under the name Steady) is an open-source vulnerability assessment tool developed by the SAP security research team in order to address this issue. Its code-centric approach, combining both static and dynamic analysis allows for scanning Python and Java applications and to detect, assess and mitigate the discovered vulnerabilities. Despite its relative youth (first early prototype dating from 2015), the tool has matured enough to be officially recommended for all applications at SAP in 2017 and as of today has accumulated over 1M+ scans of around 1000 projects.

# Implementation

Built upon the theoretical work further explained in Appendix (VIII), the assessment tool is written in Java JDK8 and with a fully dockerized architecture that comprises the following server side components:

{{< image src="/img/vulas_docker_architecture.png" alt="Hello Friend" position="center" >}}

- **frontend-apps**: OpenUI5 front-end built for users to view different test results (**app** to generate a bill of material report, **a2c** to build call graphs, etc...). On this interface, users will be able to display all vulnerabilities discovered in their package, the dependencies affected as well as the reachable call graphs.

- **frontend-bugs**: OpenUI5 front-end for viewing results of scans (required when the AST method  previously described cannot yield a definitive result).
    
- **rest-backend** Springboot backend which interfaces with the vulnerability database and serves an API for all other services in the architecture.
    
- **rest-lib-utils** Springboot backend which provides utilities for scanning such as pulling repositories from the respective source code repository and getting a list of dependencies from said archive.
    
- **patch-lib-analyzer**: Java application that establishes whether a library contains a construct modified to fix a vulnerability (aka changed-construct) in its vulnerable or fixed version. 
    
- **postgreSQL**: Database storing information regarding CVEs, construct changes and application constructs.
    
- **HAProxy**: High availability load balancing mainly used to serve the application. It is important to note that all communications between containers within the private Docker network and not through a domain name so this component is only mandatory if the application is to be exposed to the Internet.


A scan can be launched by either adding a plugin to the build process (either through the **plugin-maven** or plugin-gradle) or running a compile jar to scan a specific artifact (**cli-scanner**). In order to populate the database with known vulnerabilities, the **patch-analyzer** can load a series of CVEs crawled from the NVD database and finds the appropriate commit fix for each library. 

## Migrating to the Cloud

The migration to the cloud intends to address the following concerns with the current stack:
- **Not highly available**: although the containers are redundant within each node, the deployment itself is not resilient to outage occurring on a specific node.
- **Spin up duration**: due to its reliance on automating system requirements provisioning through Ansible, if a node were to go down, the time required to spin up a new functional instance would entail high downtime.
- **Overhead cost**: each maintenance operation (database backup, smoke test) requires a specific Ansible playbook that has to be maintained and updated. At the same time, publishing new releases of the software solely relies on docker-compose and thus, is quite complicated.
- **Non reproducible heterogeneous infrastructure**: due to an UI based infrastructure creation, and each machine serving different purposes.


### Overview of choices and challenges

Since the application was not developed with the intention of being used in the k8s/cloud native environment, the migration to the cloud requires a lot of technical choices and challenges that will detailed in the following sections:
- [Database management](../eclipse_database/): migrating the vulnerability-assessment-tool to k8s
- [Core components](../eclipse_core/): transitioning all the components vital to having a functional deployment of the vulnerability-assessment-tool
- [Support components](../eclipse_support/): adapting components destined to serve the tool over the Internet
- [Monitoring stack](../eclipse_monitoring/): translating the monitoring and logging functionality stack to a more cloud native version. 
- [Security considereations](../eclipse_security/): improving the deployment's security
- [Infrastructure provisioning](../eclipse_infrastructure/): creating the underlying machines and networking components required for running k8s.
