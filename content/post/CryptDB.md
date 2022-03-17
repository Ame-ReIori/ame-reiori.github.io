---
title: "CryptDB"
date: 2022-03-03T14:41:45+08:00
typora-root-url: ./
draft: false
---

# Problem Description

The paper considers the situation where users make a query to database on an online application. Different with former papers (meaning that according to the paper read formerly), we think that the data stored in database belong to users and want to avoid the database from learning any knowledge of these data. Then, the proposed schema is having the database store encrypted data.

However, the paper consider an adversary outside that wants to thieve the data in database. The adversary may have access to database, snoop on private data, or even access all data on disk and in memory. The problem is how to avoid the theft. Considering the cost on system deployment and client-side computation, the paper presents CryptDB, a system that explores an intermediate design point to provide confidentiality for applications that use database management systems, to solve it. 

There are two threats which CryptDB should address.

* If the adversary is a curious database administrator (DBA) , how to avoid him from learning private data.
* If the adversary gets complete control of application and servers, how to avoid him from learning private data.

In the first case, CryptDB can prevent DBA from learning any private data through some encryption schema; In the second case, **CryptDB cannot provide any guarantees for users that are logged into the application during an attack, but can still ensure the confidentially of logger-out users' data**.

It seems that CryptDB provides a good schema to solve problems in first case. However, there are still several problems, including computation and communication cost. Besides, CryptDB doesn't provide a complete schema for the second case.

In this blog, we may follow CryptDB's idea and make a sight on how it solves these problems.

## Challenages and main idea

There are two challenages when solving the threats.

* The first lies in the tension between minimizing the amount of confidential information revealed to the DBMS server and the ability to efficiently execute a variety of queries.
* The second is to minimize the amount of data leaked when an adversary compromises the application server in addition to the DBMS server.

CryptDB addresses these challenages with three key ideas:

* The first is executing SQL queries over encrypted data with SQL-aware encryption. Different private operations can be implemented with different encryption schemas. For example, SUM can be implemented with homomorphic encryption.
* The second is adjustable query-based encryption. I think it is the same as the first idea. We customize each operation with different encryption schema and encrypt data with these schemas, then we should have a scheduler make a reasonable plan for adjusting operations with schema.
* The third is chain encryption keys to user password. Actually, it is a kind of access control.

# System Overview

The construction is as followed. CryptDB add database proxy to the origin system, which encrypts and decrypts all data, and changes query operations. There is some strategies when encrypting data due to the second threat. Then, we will make a sight on  what the security and brief approach is in two threats.

![image-20220304162347255](/img/image-20220304162347255.png)

## Threat 1

Threat 1 is the situation where the adversary has full access to the data stored in the DBMS server. The paper consider the adversary to be passive, or we say, semi-honest. CryptDB protects data content and names of columns and tables, but it doesn't protect overall table structure, the number of rows, the type of columns, or the approximate size of data in bytes.

CryptDB combines SQL-aware encryption and adjustable query-based encryption, making the former one adjust dynamically to the query presented.

## Threat2

In this threat, the adversary compromises data proxy, meaning that he can get the encryption key. I think CryptDB wants to protect data of any user. However, now it can only protect data of logged-out users due to its approach. CryptDB encrypts different data item with different keys. Data of inactive users would be encrypted with keys not available to the adversary.

# Query Over Encrypted Data

CryptDB combines SQL-aware encryption and adjustable query-based encryption. The former enables us to apply different operations on encrypted data, while the later enables us to schedule the encryption schema.

## SQL-aware encryption

### Random (RND)

It is very simple. Using AES-CBC or other IND-CPA encryption schema can make the ciphertext seemed to be random. In CryptDB, they use Blowfish for integer values and AES for others.

### Deterministic (DET)

DET, which should be a PRP, maps the same value to the same ciphertext, allowing the server to perform equality checks, which means it can perform selects with equality predicates, equality joins, GROUP BY, COUNT, DISTINCT, etc. In CryptDB, they use AES with a variant of the CMC mode.

### Order-preserving encryption (OPE)

OPE remains value's order on ciphertext. For example, if $x < y$, then $OPE_{k}(x) < OPE_{k}(y)$. Therefore, the server can perform ORDER BY, MIN, MAX, SORT, etc.

### Homomorphic encryption (HOM)

Like OPE, HOM allows the server to perform some arithmetic operations. For example, With Paillier, an additive homomorphic encryption, we have $HOM_{k}(x) \cdot HOM_{k}(y) = HOM_{k}(x+y)$.

### Join (JOIN and OPE-JOIN)

Join can be classified as equi-join and range join. DET can address the former one. However, due to CryptDB's approach for threat 2, the columns joined may not be encrypted with the same key. The paper proposes a new cryptographic primitive, called JOIN-ADJ (*adjustable join*), which allows the DBMS server to adjust the key of each column at runtime.

The main aim of JOIN-ADJ is converting columns encrypted with different keys to the ones encrypted with the same key. It can be implemented with power. The main idea is as followed:
$$
(x^k)^\frac{k'}{k} = x^{k'}
$$
CryptDB gives the construction as followed:
$$
JOIN-ADJ_{K}(v) = P^{K \cdot PRF_{K_0}(v)}
$$
where $K$ is the key for columns or tables and $K_0$ is the master key which may be different asccording to users. And the following equation shows how to convert $JOIN-ADJ_{K}(v)$ to $JOIN-ADJ_{k'}(v)$:
$$
(JOIN-ADJ_{K}(v))^{\Delta K} = (JOIN-ADJ_{K}(v))^{\frac{K'}{K}} = (P^{K \cdot PRF_{K_0}(v)})^{\frac{K'}{K}} = P^{K' \cdot PRF_{K_0}(v)}
$$
By the way, with elliptic curve, the computation and storage cost can be decreased. CryptDB implements the functionality with following steps:

1. The proxy calculates $\Delta K = \frac{K'}{K}$, then sends it to the DBMS server
2. The DBMS server uses UDFs to update the corresponding column

### Word search (SEARCH)

SEARCH makes CryptDB support LIKE operator. And the authors only implement the former schema, and support full-word search.

## Adjustable query-based encryption

Each encryption schema mentioned above is suitable for a specific situation. However, a query can be complex, meaning that it may contain variety of operations. It is hard to use a single schema when processing a private query. The paper proposed adjustable query-based encryption to dynamically choose the encryption schema.

Its main idea is to encrypt value each data item in one or more onions: that is, each value is dressed in layers of increasingly stronger encryption, like

![image-20220316144121277](/img/image-20220316144121277.png)

For example, using Onion Eq, users can do equality selection and equality join, while the data looks random if an adversary get the ciphertext. For each layer of each onion, the proxy uses the same key for encrypting values in the same column while using different keys for different tables, columns, onions and onion layers. The adjustable query-based encryption works like onion. Here I'll use the description in paper.

> Given a predicate *P* on column *c* needed to execute a query on the server, the proxy first establishes what onion layer is needed to compute *P* on *c*. If the encryption of *c* is not already at an onion layer that allows *P*, the proxy strips off the onion layers to allow *P* on *c*, by sending the corresponding onion key to the server. The proxy never decrypts the data past the least-secure encryption onion layer.

The implementation of adjustable query-based encryption in CryptDB is using UDF. When needed, proxy will make a query like following and have database execute it.

```sql	
UPDATE Table1 SET C2-Ord = DECRYPT_RND(K, C2-Ord, C2-IV)
```

In the above query, `C2-Ord` is Onion Ord while `K` is the key and `C2-IV` is the random vector used in RND.

## Executing over encrypted data

Query execution over encrypted data can be seemed as rewriting the query and having the database execute it with UDF. It can be clearly explained by an example.

### Read

Consider the read query `SELECT ID FROM Employees WHERE Name=‘Alice’`. The proxy generates two query.

```sql
UPDATE Table1 SET C2-Eq = DECRYPT RND(KT1,C2,Eq,RND, C2-Eq, C2-IV)
SELECT C1-Eq, C1-IV FROM Table1 WHERE C2-Eq = x7..d
```

The first performes the adjustable query-based encryption while the second retrieves the encrypted result. Finally, the proxy decrypts the results from the server using keys $K_{T1,C1,Eq,RND}, K_{T1,C1,Eq,DET}, $ and $K_{T1,C1,Eq,JOIN}$, obtains the result, and returns it to the application. By the way, the database looks like the following.

![image-20220317193838468](/img/image-20220317193838468.png)

### Write

Write operations include INSERT, UPDATE, DELETE. The proxy applies similar processing on INSERT and DELETE operations while there are some special points when performing UPDATE. However, due to CryptDB's adjustable query-based encryption, it cannot  perform HOM and OPE simultaneously. Therefore, if a column is used in comparisons after it is incremented, the solution is to replace the update query with two queries: a SELECT of the old values to be updated, which the proxy increments and encrypts accordingly, followed by an UPDATE setting the new values. But the strategy may not work well when meeting big data.

## Improving security and performance

The paper proposes some optimizations. I think these are some improvements on implementation, not theory. But...maybe it's my fault. The improvements can be described as followed. The first four are improvements on security, and the last three are the ones on performance. The main idea of some improvements can be easily understand with subtitle, which will not be introducede below.

* **Minimum onion layers.**
* **In-proxy processing.**
* **Training mode.** CryptDB provides a training mode, which allows a developer to provide a trace of queries and get the resulting onion encryption layers for each field, along with a warning in case some query is not supported.
* **Onion re-encryption.**
* **Developer annotations.**
* **Known query set.**
* **Ciphertext pre-computing and caching**.

# MULTIPLE PRINCIPALS

