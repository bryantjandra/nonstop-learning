****
## Introduction

**Data is very important in developing applications nowadays.**  As users interact with a certain app, they both **read** the stored data, and they **generate** more data. 

As the volume of data increases for an application, it *needs* to be distributed across multiple machines, because it is not sufficient to store everything into 1 system. We thus **need multiple storage or processing systems.**

- And so, we need to be able to figure out which storage/processing systems are the most appropriate for the certain task at hand. This can be difficult as each decision has its **pros** and **cons**.


*Data-Intensive Application*: Data management is the primary challenge when developing the app. These applications typically need to do the following:

1. **Database**: Able to store data such that it can be read in the future. 
2. **Caches**: Able to remember the result of an expensive operation, speeding up reads in the future.
3. **Search Indexing**: Able to allow users to search for data by certain keywords. 
4. **Stream Processing**: Able to handle events as soon as they occur. 
5. **Batch Processing**: Able to periodically handle a large amount of accumulated data.


We now delve deep into several contrasting concepts and their trade-offs in data-intensive applications:

## Operational Vs. Analytical Systems

> NOTE: *Business Analysts* & *Data Scientists* have some practices in common:
> 1. They perform analytics --> Look at the data that the users and backend services have generated.
> 2. They generally do not modify this data.


**Operational Systems**: The databases and backend services that keep an application running day-to-day. They hold the live source data and they serve the interactive workload.

This is the application code that reads and writes individual records in real time in response to actions by users / services, such as placing an order, sending a message, or updating a profile.

1. **Point Query** --> Looks up a small number of records by a key. Usually these queries are fixed and baked into the code.
2. These records are inserted / updated / deleted based on the user's actions --> Because of this interactiveness, the **access pattern** for operational systems is known as **OLTP** (Online Transaction Processing).  


**Analytical Systems**: Contain a read-only copy of the data from **operational systems**. This read-only copy is optimized for the types of *data processing* that are needed for *business analysts* and *data scientists*.

1. **Analytical Query** --> Scans over a huge number of records and calculates aggregate statistics (summing, averaging, etc). Usually, these queries are more customizable.
2. The data queried will be used for BI (business intelligence) and this **access pattern** for analytical systems is known as **OLAP** (Online Analytical Processing)


Comparing the characteristics of Operational (OLTP) and Analytical (OLAP) systems:

1. **Main Read Pattern**
	- OLTP: Point Query
	- OLAP: Analytical Query

2. **Main Write Pattern**
	- OLTP: Insert / update / delete individual records
	- OLAP: Bulk import (ETL) or event stream

3. **Human User Example**
	- OLTP: User of mobile applications
	- OLAP: Business analyst

4. **Machine Use Example**
	- OLTP: Checking if something is authorized
	- OLAP: Detecting fraud patterns

5. **Query Volume**
	- OLTP: Lots of small queries
	- OLAP: Few queries, with each being complex

6. **Data Represents**
	- OLTP: Latest state of data
	- OLAP: History of events that happened over time.

7. **Dataset Size**
	- OLTP: GB to TB
	- OLAP: TB to PB


**Product Analytics / Real-Time Analytics**: Systems that are like analytical systems (for analytical workloads) but embedded into user-facing products. E.g. Clickhouse

1. Ingest data in real-time.
	- CONTRAST WITH OLAP: Ingest data in batches.
2. Optimized for low-latency query responses (very fast). 
	- CONTRAST WITH OLAP: Optimized for high-throughput (how much total data or how many queries a system can process per unit time) query processing.


### Data Warehouses & Data Lakes

**Data Warehousing**: Companies stopped using their OLTP systems for analytics purposes and they chose to run the analytics on a separate database instead. This separate DB is known as a *data warehouse*. It often uses **a relational data model** that is queried through SQL. 

Usually *undesirable* for business analysts and data scientists to directly query OLTP systems:

1. **Data Silos**: The data of interest may be spread across multiple different operational systems, making it difficult to access the data of interest in a single query. 
2. The **schema** used for OLTP systems are less well suited for analytics. 
3. Analytical queries can be quite expensive, thus degrading the performance for other users as well. 
4. **Security / Compliance**, causing these OLTP systems to reside in another network where analysts can't directly access.


*Data warehouses* allow analysts to query however much they want and please. There's usually just **a single** data warehouse, as compare to **hundreds** of operational databases. This is considered good practice to have each operational system have its own DB (more later on). 
 
They contain a read-only copy of the data from **all the various OLTP systems**. The data is extracted from all these OLTP systems using either a periodic data dump / continuous stream updates --> transformed --> cleaned --> loaded into the data warehouse (**ETL**).


**Data Lakes**: Centralized data repository that holds a copy of **any data that might be useful for analysis**. 

**However**, data lakes are more **flexible**. They can contain *text, images, videos, feature vectors,* etc. It contains data in the *raw* form produced by the operational systems, without the transformation into a relational data warehouse schema!

Data lakes are suited for these use cases:
1. **Transforming data into a from that is suitable for training an ML model** --> Turning the rows and columns of a DB table into a vector or matrix of numerical values (features). 
2. **Using NLP techniques** on textual data to try to extract structured information from it. 

**Advantage of Data Lakes**: Each consumer of the data can transform the raw data into the form that best suits their needs.



### Systems of Record & Derived Data

**System of Record**: Source of truth. Whenever any new data comes in, it is first written here. *If there is any discrepancy between another system and the system of record, the value in the system of record is the correct one.*

- Operational services may contain a **mixture of systems of record** and **derived data systems**

**Derived Data Systems**: Resulting data of taking existing data from another system and transforming it or processing it in some way. If you lose derived data, you can recreate it from the original source. E.g. Cache. 

- Analytical systems are usually **derived data systems** as they are consumers of data created elsewhere.

> The distinction between a system of record and a derived data system depends not on the tool, but on the way you use it in your application.

**IMPORTANT**: When the data in one system is derived from the data in another, you need a process for updating the derived data when the original in the system of record changes!

**Unfortunately**, most systems out there do not support this that well!




## Cloud Vs. Self-Hosting

Self-Hosting --> **Do it** if the thing is a core competency & offers a competitive advantage. 
Cloud / External Vendor --> **Do it** if the thing is not a core competency. 

*Should you use cloud provider / self-host?*
- It depends,
	- If we need a system that we don't know how to deploy / operate, then **just use a cloud provider.** 
	- If your dataset is so large that querying it quickly requires significant computing resources, **using a cloud provider can save money.** This is because we can return the unused resources to the provider rather than leaving them idle.
	- If we need a lot of customizations and control, it would be **advisable to just self-host.** 

### Cloud-Native System Architecture

Definition: An architecture that is designed to take advantage of **cloud services.**

Advantages of a cloud-native system architecture:
1. Better performances on the same hardware
2. Faster recovery from failures
3. Being able to quickly scale computing resources to match the load

The **key idea of cloud services** is not only to se the computing resources managed by your OS, but to also build upon *lower-level cloud services* to create higher-level services. 

Example:
- **Object storage services (Amazon S3, Azure Blob Storage) store large files.** They build upon the typical filesystem by automatically distributing the data across many machines, so we don't have to worry about running out of disk space on a single machine. 
- **Snowflake** (cloud based analytical DB / data warehouse) builds upon object storage services like S3 for data storage.

## Distributed Vs. Single Node Systems

**Distributed Systems**: System where several machines are communicating via a network. Each process participating in a distributed system is called a *node*.

*Why use distributed systems?*
1. **Inherent distribution**: system is by default distributed if an application involves >= 2 interacting users as the communication between their two devices will have to occur via a network. 
2. **Requests between cloud services**: e.g. Microservices --> where data is stored in one service needs to be processed in another.
3. **Fault tolerance / High availability**: Using multiple machines helps our application to continue working even if one machine fails (*Redundancy*). 
4. **Scalability**: Using multiple machines to help spread out load. 
5. **Latency**: Using multiple servers in various regions worldwide, so that each user can be served from a server that is geographically closer to them, reducing latency. 
6. **Elasticity**: Able to scale up or down to meet resource demand --> more cost effective.
7. **Specialized hardware**: Different parts of the system can take advantage of different types of hardware to match their workload (e.g. Object store (S3) may use certain machines with many disks but few CPUs, whereas a data analysis system may use machines with many CPUs but few disks).
8. More: **Legal Compliance** & **Sustainability**

**Issues with distributed systems**:
1. **Failure**: Possibility of failure for every request and API call as it is done over the network.
2. **Speed**: Making a call to another service is vastly slower than just doing the thing on your own machine.
3. **Troubleshooting**: Difficult to troubleshoot. But tools like **OpenTelemetry, Zipkin, & Jaeger** track which client called which server for which operation and how long each call took.
4. **Data Consistency**: If each service has its own DB, then maintaining data consistency across every service can be tricky. 

Thus, performing a task on a single machine is often much simpler and cheaper than setting up a distributed system. 

As CPUs, memory, and disks have grown larger, faster, and more reliable --> When combined with **single-node DBs** (DuckDB, SQLite, KuzuDB), many workloads can now run on a single node.



### Microservice & Severless

*Most common way to distribute a system:* Divide the system into **clients** and **servers**, a.k.a *Microservices Architecture*.

A **Microservices Architecture** has one-well defined purpose, where each individual service exposes an API that can be called by clients, and each individual service is managed by 1 team. 

Advantages of a **Microservices Architecture**:
- **Independence**: Each service can be updated independently without affecting other services.
- **Hardware Optimization**: Each service can be assigned only the hardware/resources it needs.
- **Performance**: Each service has its own DB so their queries don't affect the performance of the queries of other services. 

Cons of a **Microservices Architecture**:
- **Complexity**: 
	- Testing a service requires every other service to be on. 
	- Each service requires infra for deploying new releases, adjusting the allocated hardware resources to match the load, collecting logs, monitoring service health, and etc.
	- APIs can be challenging to evolve. 

Overall, microservices **allows different teams to make progress independently** without having to coordinate with each other. 

**This is useful in a large company,** but *unnecessary* overhead for a smaller company. 


**Serverless / FaaS**: *Outsource the management of infra to a cloud vendor.*

The cloud vendor will automatically allocate and free hardware resources as needed, depending on the incoming requests to your service. 

> Many vendors impose a time limit on function execution and they may limit runtime environments, causing services to suffer from slow start times when a function is first invoked.




