---
type: posts
title: Unusual Arbitrary Attribute
author: Amutheezan Sivagnanam
category: Internship
date: 2016-10-26
last_modified_at: 2016-11-08
tags:
- unusual attribute

---

This content is based on the issue I faced while doing analyzing with Arbitrary attribute of HL7. It is common for cases
where arbitrary attributes are similar to the case of HL7

When we are fetching data in WSO2 **DAS** through **ESB** from HAPI test-panel, we are also getting arbitrary attributes
related to HL7 Messaging addition to existing attributes related to message flow. These arbitrary attributes are for the
content of the message passed, it contains a detailed classification of each element of HL7 messages that were
transmitted.

For the better use case of HL7 messaging we need to analyze these messages by indexing these arbitrary fields. We can
add this arbitrary field as a normal or generic arbitrary field. But when we are analyzing we can't directly use them.

It is because these arbitrary fields are consists of ```.``` which make the following error

```java
Caused by: org.apache.spark.sql.AnalysisException: cannot resolve '_MSH.MessageType' given input columns: 
[meta_server_name, correlation_activity_id, type, timestamp,__MSH.MessageType , content,service_name, meta_host, 
operation_name, message_direction, status];
```

To resolve this error we need to add ```**`**``` this while doing **COUNT** or **SELECT** or **INSERT** query (not
essentially require for **CREATE** schema) in spark.
