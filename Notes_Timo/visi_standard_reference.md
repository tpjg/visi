# VISI Standard — Implementation Reference

*A practitioner's guide to building VISI-compliant messages and communication.*

This document distills the normative VISI specifications (v1.6 Bijlagen 1–13, v1.7 spec) into a single reference that a seasoned programmer or AI can use to implement valid messages and protocol-correct communication. It covers the full chain: EXPRESS data model → XSD promotion → message composition → SOAP transport.

For the "why" behind VISI's design: see Appendix B.

---

## Table of Contents

1. [Overview: What VISI Is](#1-overview-what-visi-is)
2. [The File Numbering System](#2-the-file-numbering-system)
3. [Part I: The Framework Data Model (_2.exp)](#3-part-i-the-framework-data-model-_2exp)
4. [Part II: The Message Instance Model (_5.exp)](#4-part-ii-the-message-instance-model-_5exp)
5. [The Framework XML (_7.xml)](#5-the-framework-xml-_7xml)
6. [XSD Promotion: From Framework to Validating Schema](#6-xsd-promotion-from-framework-to-validating-schema)
7. [The Project-Specific Message (PSB)](#7-the-project-specific-message-psb)
8. [Message Composition](#8-message-composition)
9. [The MITT Graph: Message Sequencing](#9-the-mitt-graph-message-sequencing)
10. [Element Conditions: Field Editability Rules](#10-element-conditions-field-editability-rules)
11. [Attachments](#11-attachments)
12. [Role Transfer: Successor and Substituting](#12-role-transfer-successor-and-substituting)
13. [SOAP Communication Protocol](#13-soap-communication-protocol)
14. [Version Differences (v1.3 → v1.7)](#14-version-differences-v13--v17)

**Appendix A** — [DEMO Transaction Pattern](#appendix-a-demo-transaction-pattern)
**Appendix B** — [Design Rationale: Non-Obvious Decisions](#appendix-b-design-rationale-non-obvious-decisions)
**Appendix C** — [Reference Documents](#appendix-c-reference-documents)

---

## 1. Overview: What VISI Is

VISI is a Dutch open standard for structured, traceable communication between organizations in construction projects. It replaces unstructured email with a protocol where:

- A **framework** (raamwerk) defines the allowed communication: which roles exist, which transactions they can have, and which messages those transactions consist of.
- A **project** instantiates a framework with real people, organizations, and SOAP endpoint URLs.
- **Messages** are exchanged between exactly two persons-in-role via SOAP. Each message is self-contained: it carries not just its payload but the full context needed to place it in its transaction.
- The protocol is **asynchronous**: sending a message and receiving confirmation are two separate HTTP connections.

The standard is built on [DEMO](https://en.wikipedia.org/wiki/Design_%26_Engineering_Methodology_for_Organizations) (Design & Engineering Methodology for Organizations), a formal theory of organizational communication. This shows up in the transaction pattern: every exchange has an initiator and executor, messages alternate between them, and each message has an intention (request, promise, state, accept, reject, etc.).

---

## 2. The File Numbering System

VISI uses a numbered pipeline of files, each a transformation of the previous:

| File | Name | What it contains |
|------|------|-----------------|
| **_2.exp** | EXPRESS Part I | The VISI data model — entity type definitions (TransactionType, MessageType, RoleType, etc.) |
| **_3.xsd** | XSD from _2 | XSD to validate framework XML files. Generated from _2.exp alone. |
| **_5.exp** | EXPRESS Part II | The message instance model — how runtime instances look (MessageTemplate, PersonInRole, etc.) |
| **_7.xml** | Framework (Raamwerk) | A specific framework — concrete definitions of roles, transactions, messages for a project |
| **_9.xsd** | Promoted schema (in-memory) | XSD to validate VISI messages — generated from _2 + _5 + _7 combined |
| **_10.xsd** | Promoted schema (with namespace) | Same as _9 but serialized with the project-specific targetNamespace |

The transformation pipeline:

```
_2.exp (VISI data model)
  │
  ├──→ exp2xsd ──→ _3.xsd  (validates framework XML)
  │
  │   _5.exp (instance model)
  │     │
  │     │   _7.xml (a specific framework)
  │     │     │
  └─────┴─────┴──→ Promotor ──→ _9/_10.xsd  (validates messages for THAT framework)
```

> **Key insight:** The _3.xsd validates that a framework is structurally valid. The _10.xsd validates that messages exchanged in a project using that framework are valid. They are different schemas for different purposes.

---

## 3. Part I: The Framework Data Model (_2.exp)

The _2.exp EXPRESS schema defines the **type-level** entities — the vocabulary a framework author uses to design communication agreements. Think of these as the "class definitions" that framework XML instantiates.

Version-specific files: `_2.exp` exists per VISI version (v1.3: namespace `20110819`, v1.4: `20140331`, v1.6: `20160331`, v1.7: `20220930`).

### 3.1 Entity Reference

#### ProjectType
Defines the project container. Generally one per framework.

```
ENTITY ProjectType;
  namespace     : STRING;            -- REQUIRED. Unique URI for this framework version.
  description   : STRING;            -- REQUIRED.
  state, dateLaMu, userLaMu, language, helpInfo, code : OPTIONAL ...
  complexElements : OPTIONAL SET OF ComplexElementType;
END_ENTITY;
```

The `namespace` field is critical: it becomes the `targetNamespace` of the promoted XSD and appears in every message's `xmlns` attribute. It uniquely identifies the framework version.

#### RoleType
Defines a participant type (e.g., "Contractor", "Client", "Waiter").

```
ENTITY RoleType;
  description : STRING;              -- REQUIRED. Human-readable role name.
  responsibilityScope, responsibilityTask,
  responsibilitySupportTask, responsibilityFeedback : OPTIONAL STRING;
END_ENTITY;
```

#### TransactionType
Defines a binding agreement between exactly two roles.

```
ENTITY TransactionType;
  description : STRING;              -- REQUIRED.
  initiator   : RoleType;            -- REQUIRED. The role that starts the transaction.
  executor    : RoleType;            -- REQUIRED. The role that responds.
  appendixTypes : OPTIONAL SET OF AppendixType;
END_ENTITY;
```

A transaction is always between exactly two roles. The initiator sends the first message; then messages alternate between initiator and executor.

#### MessageType
Defines the structure of a message: which data elements it contains.

```
ENTITY MessageType;
  description       : STRING;          -- REQUIRED.
  appendixMandatory : OPTIONAL BOOLEAN; -- If true, at least one attachment required.
  complexElements   : OPTIONAL SET OF ComplexElementType;
  appendixTypes     : OPTIONAL SET OF AppendixType;
END_ENTITY;
```

#### MessageInTransactionType (MITT)
The **central linking entity**. A MITT places a specific MessageType at a specific step in a TransactionType's flow.

```
ENTITY MessageInTransactionType;
  message                          : MessageType;              -- REQUIRED.
  transaction                      : TransactionType;          -- REQUIRED.
  initiatorToExecutor              : OPTIONAL BOOLEAN;         -- Direction of this message.
  firstMessage                     : OPTIONAL BOOLEAN;         -- Can this start a (sub)transaction?
  previous                         : OPTIONAL SET OF MITT;     -- Predecessor steps.
  openSecondaryTransactionsAllowed : OPTIONAL BOOLEAN;         -- Can spawn child transactions?
  transactionPhase                 : OPTIONAL TransactionPhaseType;
  conditions                       : OPTIONAL SET OF MITTCondition;
  group                            : OPTIONAL GroupType;
  appendixTypes                    : OPTIONAL SET OF AppendixType;
END_ENTITY;
```

The `previous` field creates a directed graph of message steps. See [§9 The MITT Graph](#9-the-mitt-graph-message-sequencing).

#### MessageInTransactionTypeCondition
Cross-transaction ordering constraints.

```
ENTITY MessageInTransactionTypeCondition;
  sendAfter  : OPTIONAL SET OF MITT;  -- ALL must have been sent before this MITT is available.
  sendBefore : OPTIONAL SET OF MITT;  -- NONE may have been sent before this MITT is available.
END_ENTITY;
```

#### ComplexElementType
Groups fields into logical sections (like form chapters). Can nest.

```
ENTITY ComplexElementType;
  description     : STRING;
  complexElements : OPTIONAL SET OF ComplexElementType;  -- Nested groups.
  simpleElements  : OPTIONAL SET OF SimpleElementType;   -- Fields in this group.
  minOccurs       : OPTIONAL INTEGER;  -- Table row minimum (v1.6+).
  maxOccurs       : OPTIONAL INTEGER;  -- Table row maximum (v1.6+).
END_ENTITY;
```

When `minOccurs`/`maxOccurs` are set on a nested ComplexElementType, it becomes a repeating table.

#### SimpleElementType
A single typed data field.

```
ENTITY SimpleElementType;
  description     : STRING;
  userDefinedType : UserDefinedType;  -- REQUIRED. The field's type and restrictions.
END_ENTITY;
```

#### UserDefinedType
Specifies the data type and optional validation restrictions.

```
ENTITY UserDefinedType;
  description    : STRING;
  baseType       : STRING;           -- BOOLEAN | DATE | DATETIME | TIME | DECIMAL | INTEGER | STRING
  xsdRestriction : OPTIONAL STRING;  -- XSD restriction fragment (pattern, enumeration, etc.)
END_ENTITY;
```

#### ElementCondition
Controls field editability across message steps. See [§10](#10-element-conditions-field-editability-rules).

```
ENTITY ElementCondition;
  description      : STRING;
  condition         : STRING;                            -- "FREE" | "FIXED" | "EMPTY"
  complexElements   : OPTIONAL SET [0:2] OF ComplexElementType;
  simpleElement     : OPTIONAL SimpleElementType;
  messageInTransaction : OPTIONAL MITT;
END_ENTITY;
```

#### Other types

- **AppendixType** — Attachment metadata definition, with optional complexElements.
- **TransactionPhaseType** — DEMO phase label (e.g., "Requested", "Promised", "Accepted").
- **GroupType** — Grouping mechanism for attachments.
- **PersonType**, **OrganisationType** — Type definitions for person/org instances, with optional complexElements for custom fields.

---

## 4. Part II: The Message Instance Model (_5.exp)

The _5.exp EXPRESS schema defines the **instance-level** entities — the objects that appear in actual VISI messages and the Project-Specific Message (PSB). Think of these as "instances of the types defined in _2.exp."

> **Important terminology:** "Template" in VISI jargon means **"instance that lives in a message"**, not "blueprint." This is a linguistic choice that initially confuses everyone, but once you see it, the entire structure falls into place.

### 4.1 Entity Reference

#### MessageTemplate
The actual message sent in a transaction. **This is the top-level object in a VISI message.**

```
ENTITY MessageTemplate;
  identification                 : STRING;                         -- REQUIRED. Unique ID.
  dateSend                       : DATETIME;                       -- REQUIRED.
  dateRead                       : OPTIONAL DATETIME;
  initiatingTransactionMessageID : OPTIONAL STRING;                -- Links to parent transaction.
  initiatorToExecutor            : BOOLEAN;                        -- REQUIRED. Direction.

  messageInTransaction : MessageInTransactionTemplate;             -- REQUIRED. MITT instance.
  transaction          : TransactionTemplate;                      -- REQUIRED. Transaction instance.
  template             : ComplexElementTemplate;                   -- REQUIRED. Message-level custom fields.
END_ENTITY;
```

The three embedded entities are all **required**. This is the normative basis for message self-containment.

> **Implementation note:** The `template` field on MessageTemplate wraps the message's custom ComplexElementTemplate — it is **not** the message payload fields themselves. The actual data fields (from the framework's `MessageType.complexElements`) are serialized as direct children of the message element in the promoted XSD, not inside a `<template>` wrapper. The `template` here is the EXPRESS model's way of saying "this message has a complex element structure"; the promoted XSD flattens this into named child elements.

#### MessageInTransactionTemplate
Runtime record of a MITT step. Minimal — just identification and timestamps.

```
ENTITY MessageInTransactionTemplate;
  identification : STRING;
  dateSend       : OPTIONAL DATETIME;
  dateRead       : OPTIONAL DATETIME;
END_ENTITY;
```

#### TransactionTemplate
Runtime instance of a TransactionType. Links the two participants.

```
ENTITY TransactionTemplate;
  number      : INTEGER;               -- REQUIRED. Sequence number.
  name        : STRING;                -- REQUIRED.
  description : STRING;                -- REQUIRED.

  initiator : PersonInRole;            -- REQUIRED. The person-in-role initiating.
  executor  : PersonInRole;            -- REQUIRED. The person-in-role executing.
  project   : ProjectTypeInstance;     -- REQUIRED. The project context.
END_ENTITY;
```

#### PersonInRole
**The pivotal entity.** Binds a person to a role within an organization.

```
ENTITY PersonInRole;
  successor    : OPTIONAL PersonInRole;      -- Permanent replacement (see §12).
  substituting : OPTIONAL PersonInRole;      -- Temporary delegate (see §12).

  contactPerson : PersonTemplate;            -- REQUIRED.
  organisation  : OrganisationTemplate;      -- REQUIRED.
  role          : RoleTemplate;              -- REQUIRED.
END_ENTITY;
```

#### OrganisationTemplate
```
ENTITY OrganisationTemplate;
  name          : STRING;
  abbreviation  : STRING;
  contactPerson : PersonTemplate;            -- REQUIRED. Organisation's contact person.
  template      : ComplexElementTemplate;    -- Organisation-specific fields (including SOAPServerURL).
END_ENTITY;
```

#### PersonTemplate, RoleTemplate
Simple data carriers:
```
PersonTemplate:   userName, name, template (ComplexElementTemplate)
RoleTemplate:     name, description
```

#### ProjectTypeInstance
```
ENTITY ProjectTypeInstance;
  name, description : STRING;
  template : ComplexElementTemplate;         -- Project-specific fields (SOAPProtocol, SOAPServerURL, etc.)
END_ENTITY;
```

#### AppendixTemplate
File attachment instance:
```
ENTITY AppendixTemplate;
  name, fileLocation, fileType, fileVersion : STRING;
  checksum : STRING;                         -- v1.7+ only. Absent in v1.6 and earlier.
  message        : MessageTemplate;          -- REQUIRED. Links attachment to its message.
  appendixGroup  : OPTIONAL AppendixGroup;
  template       : ComplexElementTemplate;
END_ENTITY;
```

#### ComplexElementTemplate
Wraps a SimpleElementVirtual for custom data fields. This is the leaf of the template tree.

```
ENTITY ComplexElementTemplate;
  template : SimpleElementVirtual;
END_ENTITY;
```

---

## 5. The Framework XML (_7.xml)

A framework XML is a `visiXML_VISI_Systematics` document that instantiates the _2.exp types. It is the **communication agreement** for a project: which roles exist, which transactions they can have, which messages those transactions consist of, and what data each message carries.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<visiXML_VISI_Systematics xmlns="http://www.visi.nl/schemas/20160331">

  <ProjectType id="ProjectType1">
    <namespace>http://www.visi.nl/schemas/20160331/TestFramework</namespace>
    <description>Restaurant ordering system</description>
    <complexElements>
      <ComplexElementTypeRef idref="CeSOAP"/>
    </complexElements>
  </ProjectType>

  <RoleType id="ober">
    <description>Waiter</description>
  </RoleType>

  <RoleType id="klant">
    <description>Customer</description>
  </RoleType>

  <TransactionType id="t1_OpnameBestelling">
    <description>T1 Taking an order</description>
    <initiator><RoleTypeRef idref="ober"/></initiator>
    <executor><RoleTypeRef idref="klant"/></executor>
  </TransactionType>

  <MessageType id="msgWiltuDeKaartZien">
    <description>Would you like to see the menu?</description>
    <complexElements>
      <ComplexElementTypeRef idref="CeMenuKaart"/>
    </complexElements>
  </MessageType>

  <MessageInTransactionType id="BerichtInTransactie1">
    <message><MessageTypeRef idref="msgWiltuDeKaartZien"/></message>
    <transaction><TransactionTypeRef idref="t1_OpnameBestelling"/></transaction>
    <initiatorToExecutor>true</initiatorToExecutor>
    <firstMessage>true</firstMessage>
    <!-- No previous: this is the start message -->
  </MessageInTransactionType>

  <!-- ... more MITTs, ComplexElementTypes, SimpleElementTypes, etc. -->
</visiXML_VISI_Systematics>
```

The framework XML is validated against _3.xsd (generated from _2.exp). Every `id` attribute here becomes an XML element name in the promoted _10.xsd.

---

## 6. XSD Promotion: From Framework to Validating Schema

### What promotion does

The Promotor takes three inputs (_2.exp + _5.exp + _7.xml) and produces a project-specific XSD that validates VISI messages. The `id` attributes from the framework XML become concrete element names in the XSD.

### The algorithm (conceptual)

1. **Parse _2.exp** — Understand entity types, their attributes, which are optional, and their base types.
2. **Parse _5.exp** — Understand the template/instance structure for messages.
3. **Parse _7.xml** — Read the concrete framework definitions.
4. **For each entity in _7.xml**, resolve its type from _2.exp. For example, `<MessageType id="msgWiltuDeKaartZien">` in _7.xml maps to the `MessageType` entity in _2.exp.
5. **Generate XSD** where:
   - Root element is `<visiXML_MessageSchema>` (a `<xs:choice maxOccurs="unbounded">` over all entity types).
   - Element names come from _7.xml `id` attributes (e.g., `msgWiltuDeKaartZien`, `t1_OpnameBestelling`).
   - Element structure comes from the _5.exp template definitions (e.g., `MessageTemplate` requires `identification`, `dateSend`, `initiatorToExecutor`, `messageInTransaction`, `transaction`, `template`).
   - Field types and restrictions come from `UserDefinedType.baseType` and `xsdRestriction` in _2.exp.

### The namespace

The promoted XSD's `targetNamespace` comes from `ProjectType.namespace` in the framework XML:

```
Framework XML:    <namespace>http://www.visi.nl/schemas/20160331/TestFramework</namespace>
Promoted XSD:     targetNamespace="http://www.visi.nl/schemas/20160331/TestFramework"
Messages:         xmlns="http://www.visi.nl/schemas/20160331/TestFramework"
```

This means messages validated against framework version A won't accidentally validate against version B — their namespaces differ.

### The promoted XSD root structure

```xml
<xs:schema targetNamespace="http://www.visi.nl/schemas/20160331/TestFramework">
  <xs:element name="visiXML_MessageSchema">
    <xs:complexType>
      <xs:choice maxOccurs="unbounded">
        <!-- Every entity from _7.xml becomes an element here -->
        <xs:element ref="msgWiltuDeKaartZien"/>
        <xs:element ref="t1_OpnameBestelling"/>
        <xs:element ref="PersonInRole"/>
        <xs:element ref="StandardOrganisationType"/>
        <xs:element ref="StandardPersonType"/>
        <xs:element ref="klant"/>     <!-- Role id from framework -->
        <xs:element ref="ober"/>      <!-- Role id from framework -->
        <!-- etc. -->
      </xs:choice>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

The `<xs:choice maxOccurs="unbounded">` is not "lax validation" — it's the serialized form of the EXPRESS embedded-object model. Embedded entities are floated to root level, and parents reference them via `<XRef idref="..."/>`.

### XSD element ordering (xs:sequence)

The promoted XSD uses `xs:sequence` for child elements of each entity type, meaning **child element order matters for validation**. An implementation that builds message XML must:

1. Parse the promoted XSD to extract the declared element order for each named type.
2. After building the message content, reorder child elements to match `xs:sequence` order.
3. Elements not declared in the XSD sequence are appended at the end.

This is easy to miss — building elements in the wrong order produces a message that looks correct but fails XSD validation.

### XML ID validity

The `id` attribute on all elements uses `xs:ID` type, which requires the value to start with a letter or underscore (not a digit). If your instance IDs are UUIDs or numeric, you must prefix them (e.g., `id-12345` or `_12345`) to produce valid XML.

### Practical implication

Software that processes VISI messages **must** have the promoted XSD (_10.xsd) for the framework in use. Either:
- The Promotor DLL generates it at framework upload time, or
- It is uploaded alongside the framework XML, or
- The software implements its own promotion logic (rare — the Promotor is a legacy C++ DLL).

---

## 7. The Project-Specific Message (PSB)

The PSB (Project-Specifiek Bericht) bootstraps a project with real participants. It is a `visiXML_MessageSchema` document — **structurally identical to a regular message** — but contains only configuration entities, no MessageTemplate.

A PSB contains:
- **ProjectTypeInstance** — Project name, description, and technical configuration (SOAPProtocol, SOAPServerURL, SOAPCentralServerURL).
- **OrganisationTemplate(s)** — Each participating organization, with its SOAPServerURL.
- **PersonTemplate(s)** — The actual people.
- **PersonInRole(s)** — The binding of people to roles within organizations.
- **RoleTemplate(s)** — Role instances.
- **ComplexElementTemplate(s)** — Organisation/project-specific data fields.

What a PSB does **not** contain:
- MessageTemplate (no messages — it's just configuration)
- TransactionTemplate (no active transactions)
- MessageInTransactionTemplate (no MITT instances)

### PSB as source of truth

When a new transaction is started, the first message draws its embedded templates from the PSB: the two PersonInRoles (initiator + executor), their organisations, persons, roles, and the project configuration. This is **not** copying the entire PSB — it's extracting the **transitive closure** starting from the two PersonInRoles involved in this specific transaction.

### Distribution

The PSB is downloaded once from a static URI at project initialization (Bijlage 8 §3.3). Updates are distributed via a formal process with version numbers and effective dates (Bijlage 8 §5), or — in newer practice — via the meta-framework's T02 transaction.

---

## 8. Message Composition

### 8.1 The self-containment rule

Every VISI message carries the full context needed to place it in its transaction, without external lookups. This is not a convention — it's enforced by the EXPRESS model. The non-OPTIONAL fields of `MessageTemplate` require embedded `TransactionTemplate`, which requires embedded `PersonInRole`s, which require embedded `OrganisationTemplate`, `PersonTemplate`, and `RoleTemplate`.

The **transitive closure** from a single MessageTemplate:

```
MessageTemplate
├── messageInTransaction : MessageInTransactionTemplate     [required]
├── transaction : TransactionTemplate                       [required]
│   ├── number, name, description                           [required]
│   ├── initiator : PersonInRole                            [required]
│   │   ├── contactPerson : PersonTemplate                  [required]
│   │   ├── organisation : OrganisationTemplate             [required]
│   │   │   ├── contactPerson : PersonTemplate              [required]
│   │   │   └── template : ComplexElementTemplate           [required, contains SOAPServerURL]
│   │   └── role : RoleTemplate                             [required]
│   ├── executor : PersonInRole                             [required, same subtree]
│   └── project : ProjectTypeInstance                       [required]
│       └── template : ComplexElementTemplate               [required, contains project config]
└── template : ComplexElementTemplate                       [required, message-level custom fields]
```

So every message contains: the message itself, its MITT record, the transaction (with both PersonInRoles and their full organisation/person/role subtrees), and the project configuration.

> **Implementation note:** The transitive closure also includes **all MessageInTransactionTemplate elements** from the source document (PSB or previous message). These are carried forward even though they are not direct children of the MessageTemplate — they must appear at root level in the serialized XML.

### 8.2 First message in a transaction

**Source:** The locally cached PSB.

**Algorithm:**
1. User selects a TransactionType from the framework (only types where the user's PersonInRole matches the initiator role).
2. User selects the executor PersonInRole.
3. Software extracts from the PSB:
   - Initiator PersonInRole → contactPerson, organisation (with its contactPerson + complex elements including SOAPServerURL), role
   - Executor PersonInRole → same subtree
   - ProjectTypeInstance → with its complex elements (SOAPProtocol, SOAPServerURL, SOAPCentralServerURL)
4. Software creates a fresh TransactionTemplate (new id, sequential number).
5. Software creates the MessageTemplate with user-filled data fields.
6. Software creates the MessageInTransactionTemplate referencing the start MITT.
7. All embedded objects are floated to root level of `<visiXML_MessageSchema>`.
8. The result is validated against the promoted _10.xsd.

### 8.3 Continuation message (within existing transaction)

**Source:** The previously received message in this transaction.

**Algorithm:**
1. Software identifies the available next MITTs (see [§9](#9-the-mitt-graph-message-sequencing)).
2. User selects a MITT and fills in the data fields.
3. Software extracts from the **previous message** (not the PSB):
   - The TransactionTemplate — **verbatim, unchanged**. The transaction number, initiator, executor, project are frozen from the first message.
   - Both PersonInRole subtrees — carried forward.
   - The ProjectTypeInstance — carried forward.
4. Software creates a new MessageTemplate and MessageInTransactionTemplate.
5. Data fields may be carried forward, cleared, or made editable per [ElementCondition rules](#10-element-conditions-field-editability-rules).

> **Critical rule (Bijlage 8 §3.4 stap 1):** *"Het VISI bericht wordt opgemaakt door het versturende IS op basis van het ontvangen bericht (in geval van een nieuwe transactie wordt de informatie uit het projectspecifieke bericht gehaald)."* — The message is composed based on the received message; only for a new transaction is the PSB used.

This means a running transaction is **frozen on its original framework + PSB state**. If the PSB is updated mid-transaction, ongoing transactions are not affected — they carry forward their embedded context from message to message.

### 8.4 XML serialization

The promoted XSD root `<visiXML_MessageSchema>` uses `<xs:choice maxOccurs="unbounded">`. Embedded entities are serialized as root-level elements and referenced via `<XRef idref="..."/>`.

Example of a first message in the TopKoks framework:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<visiXML_MessageSchema
    xmlns="http://www.visi.nl/schemas/20160331/TestFramework"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.visi.nl/schemas/20160331/TestFramework 10.xsd">

  <!-- The message itself -->
  <msgPlaatsingBestelling id="bericht009">
    <identification/>
    <dateSend>2008-05-04T00:00:00.0Z</dateSend>
    <initiatorToExecutor>true</initiatorToExecutor>
    <messageInTransaction>
      <BerichtInTransactie7Ref idref="BiT001"/>
    </messageInTransaction>
    <transaction>
      <t1_OpnameBestellingRef idref="transactie001"/>
    </transaction>
    <ceBestelling>
      <CeBestelling id="bestelling001">
        <ceInhoudBestelling>
          <CeInhoudBestelling id="inhoudBestelling001">
            <naamGerecht>Biefstuk</naamGerecht>
            <opmerking>met mayonaise</opmerking>
          </CeInhoudBestelling>
        </ceInhoudBestelling>
      </CeBestelling>
    </ceBestelling>
  </msgPlaatsingBestelling>

  <!-- MITT instance (floated to root) -->
  <BerichtInTransactie7 id="BiT001">
    <identification/>
  </BerichtInTransactie7>

  <!-- Transaction instance (floated to root) -->
  <t1_OpnameBestelling id="transactie001">
    <number>1</number>
    <name/>
    <description/>
    <startDate>2008-05-04T00:00:00.0Z</startDate>
    <endDate>2009-05-04T00:00:00.0Z</endDate>
    <initiator><PersonInRoleRef idref="PiR002"/></initiator>
    <executor><PersonInRoleRef idref="PiR001"/></executor>
    <project><ProjectType1Ref idref="Project1"/></project>
  </t1_OpnameBestelling>

  <!-- Project (floated) -->
  <ProjectType1 id="Project1">
    <name>Project Top Koks</name>
    <description>Order-taking project</description>
    <ceSOAP>
      <CeSOAP id="ProjectGegevens">
        <sOAPProtocol>MTOM</sOAPProtocol>
        <sOAPServerURL>http://192.168.0.1/visi.wsdl</sOAPServerURL>
        <sOAPCentralServerURL>http://www.crow.nl/testcases/case001/visi.wsdl</sOAPCentralServerURL>
      </CeSOAP>
    </ceSOAP>
  </ProjectType1>

  <!-- Organisations (floated) -->
  <StandardOrganisationType id="consument">
    <name>Consument</name>
    <abbreviation>CSMT</abbreviation>
    <contactPerson><StandardPersonTypeRef idref="KeesDeVries"/></contactPerson>
    <ceOrganisatie>
      <CeOrganisatie id="OrganisatieGegevensConsument">
        <sOAPServerURL>http://192.168.0.102/specifiek_project/visi.wsdl</sOAPServerURL>
      </CeOrganisatie>
    </ceOrganisatie>
  </StandardOrganisationType>

  <StandardOrganisationType id="restaurant">
    <name>Restaurant</name>
    <abbreviation>RSTR</abbreviation>
    <contactPerson><StandardPersonTypeRef idref="PietJansen"/></contactPerson>
    <ceOrganisatie>
      <CeOrganisatie id="OrganisatieGegevensRestaurant">
        <sOAPServerURL>http://192.168.0.105/specifiek_project/visi.wsdl</sOAPServerURL>
      </CeOrganisatie>
    </ceOrganisatie>
  </StandardOrganisationType>

  <!-- Persons (floated) -->
  <StandardPersonType id="KeesDeVries">
    <userName>kvd</userName>
    <name>Kees de Vries</name>
  </StandardPersonType>

  <StandardPersonType id="PietJansen">
    <userName>pj</userName>
    <name>Piet Jansen</name>
  </StandardPersonType>

  <!-- Roles (floated) -->
  <klant id="klantRol">
    <name>Klant Rol</name>
    <description>The customer</description>
  </klant>

  <ober id="oberRol">
    <name>Ober Rol</name>
    <description>The waiter</description>
  </ober>

  <!-- PersonInRoles (floated) -->
  <PersonInRole id="PiR001">
    <contactPerson><StandardPersonTypeRef idref="KeesDeVries"/></contactPerson>
    <organisation><StandardOrganisationTypeRef idref="consument"/></organisation>
    <role><klantRef idref="klantRol"/></role>
  </PersonInRole>

  <PersonInRole id="PiR002">
    <contactPerson><StandardPersonTypeRef idref="PietJansen"/></contactPerson>
    <organisation><StandardOrganisationTypeRef idref="restaurant"/></organisation>
    <role><oberRef idref="oberRol"/></role>
  </PersonInRole>
</visiXML_MessageSchema>
```

Note how every embedded entity (transaction, project, organisations, persons, roles, PersonInRoles) is a root-level element. The message references them via `<XRef idref="..."/>`. This flat structure is the serialized form of the EXPRESS embedded-object model.

---

## 9. The MITT Graph: Message Sequencing

### 9.1 Graph structure

The MITTs in a framework form a directed graph per TransactionType:
- **Nodes:** MessageInTransactionType instances
- **Edges:** `previous` references (backward links)
- **Entry points:** MITTs where `previous` is empty (or absent) — the start message(s)
- **Subtransaction spawns:** MITTs with `firstMessage=true` AND `previous` pointing to a MITT in a **different** TransactionType

### 9.2 Availability rules

At runtime, a MITT is available (can be sent by the current user) when:

1. **At least one** `previous` MITT has been sent (OR semantics).
2. **All** `sendAfter` conditions are satisfied: every referenced MITT in every `sendAfter` set must have been sent (AND).
3. **No** `sendBefore` condition is violated: none of the referenced MITTs in any `sendBefore` set may have been sent yet (AND).
4. The direction matches: `initiatorToExecutor=true` → only the initiator can send; `false` → only the executor.

```
previous:   [A, B]      → available when A OR B has been sent
sendAfter:  [C, D]      → available when C AND D have ALREADY been sent
sendBefore: [E, F]      → available as long as NEITHER E NOR F has been sent yet
```

> **Naming clarification:** `sendBefore` is confusingly named. It means "this MITT must be sent *before* the referenced MITTs can be sent" — i.e., once the referenced MITTs are sent, this MITT is no longer available. Think of it as a "block-after" constraint: the referenced MITTs block this MITT once they fire. The spec text (MITTCondition): "may only be sent as long as the referenced MITTs have NOT yet been sent."

### 9.3 Cross-transaction references

A MITT with `firstMessage=true` can have `previous` pointing to MITTs in a different TransactionType. This is the mechanism for subtransactions:

```
Parent transaction T1:
  MITT7 (order placed) → openSecondaryTransactionsAllowed=true

Child transaction T3:
  MITT12 (cook receives order) → firstMessage=true, previous=[MITT7]
```

When MITT7 is sent in T1, MITT12 becomes available to start a new T3 instance.

### 9.4 Example: TopKoks T1 (Taking an order)

```
[MITT1] "Want to see the menu?"  ober→klant    firstMessage=true
   │
   ├──► [MITT2a] "No"               klant→ober
   ├──► [MITT2b] "No with comment"  klant→ober
   └──► [MITT3]  "Yes"              klant→ober
            │
            └──► [MITT4] "Here's the menu"  ober→klant
                    │
                    ├──► [MITT5] "Question"    klant→ober
                    │       └──► [MITT6] "Answer"  ober→klant  openSecondary=true
                    │               └──► (back to MITT5 or MITT7)
                    │
                    └──► [MITT7] "Place order"  klant→ober
                            └──► [MITT12] → starts child transaction T3
```

---

## 10. Element Conditions: Field Editability Rules

ElementConditions control whether data fields can be modified across consecutive messages within a transaction.

### Condition values

| Condition | Meaning |
|-----------|---------|
| **FREE** | Field may be modified by the sender |
| **FIXED** | Field value is locked — must carry forward from the previous message unchanged |
| **EMPTY** | Field is cleared; sender must provide a new value |

### Scope and priority

An ElementCondition can be scoped to:
- A **ComplexElement + SimpleElement** combination (most specific)
- A **ComplexElement** alone
- A **SimpleElement** alone
- A **MITT** alone (least specific)

Priority (highest wins): ComplexElement + SimpleElement > ComplexElement > SimpleElement > MITT.

### Default behavior

Without explicit conditions, the default for continuation messages is: values carry forward from the previous message (implicit FIXED behavior for fields that existed in the previous message).

---

## 11. Attachments

### Framework-level configuration

Attachments can be restricted at three levels:
1. **TransactionType.appendixTypes** — Allowed for all messages in this transaction.
2. **MessageType.appendixTypes** — Allowed for this message type across all transactions.
3. **MITT.appendixTypes** — Allowed only for this specific message step.

If `MessageType.appendixMandatory = true`, at least one attachment is required.

### Instance-level structure (AppendixTemplate)

Each attachment has:
- `name` — File name
- `fileLocation` — Original file path
- `fileType` — MIME type (e.g., `application/pdf`)
- `fileVersion` — Version string
- `checksum` — SHA256+ hash (v1.7+)
- `message` — Required reference to the MessageTemplate
- `template` — Optional ComplexElementTemplate for attachment-specific metadata

### Transport

Attachments are sent as MTOM binary parts within the SOAP envelope. The `id` of each `<Data>` element in the SOAP body must match the `id` of its corresponding AppendixTemplate in the VISI message.

---

## 12. Role Transfer: Successor and Substituting

### Successor (permanent replacement)

When a person permanently leaves a role, a `successor` PersonInRole is created. The original PersonInRole gains a `<successor>` child pointing to the replacement.

Rules:
- The original PersonInRole is **never deleted** (audit trail requirement).
- Successors form an unbroken chain: A → B → C.
- In continuation messages, the `<successor>` link is added to the PersonInRole already present in the message. The new successor's full subtree (person, organisation, role) is added to the message's root-level entities.
- The TransactionTemplate's `<initiator>` and `<executor>` references are **never changed** — they always point to the original PersonInRole. The successor chain is followed at the application level.
- **Multi-hop chains** are supported: A → B → C → D. When building a continuation message, an implementation should BFS-walk the successor chain and add all newly encountered PIR subtrees. If B already has a `<successor>` child pointing to C from a prior message, follow the chain without re-patching; only add the `<successor>` link to the **latest** PIR in the chain.

### Substituting (temporary delegation)

When a person acts on behalf of another, the delegate's PersonInRole has a `<substituting>` child pointing to the principal.

Rules:
- Multiple delegates can act for the same principal.
- The principal retains authority — delegation can be revoked.
- In the message body, the delegate's PersonInRole subtree is added alongside the original.
- The TransactionTemplate references are unchanged.

### Implementation in continuation messages

When building a continuation message where role transfer has occurred:
1. Copy the TransactionTemplate verbatim from the previous message.
2. Check if the current sender's PersonInRole has a successor or is substituting.
3. If successor: add `<successor><PersonInRoleRef idref="..."/></successor>` to the original PIR. Add the successor's full subtree (PersonInRole + person + org + role) to the message root.
4. If substituting: add the delegate's PersonInRole subtree to the message root.
5. Never modify the `<initiator>` or `<executor>` references on the TransactionTemplate.

---

## 13. SOAP Communication Protocol

### 13.1 Architecture

VISI uses direct SOAP server-to-server communication. No intermediaries:

```
Sending IS ←→ Sending SOAP Server ←→ Receiving SOAP Server ←→ Receiving IS
```

- **SOAP servers are VISI-unaware.** They cannot parse VISI message content. They only handle transport.
- **Information Systems (IS)** are VISI-aware. They compose messages, extract URLs, and validate content.
- **Protocol:** MTOM (Message Transmission Optimisation Mechanism) over HTTPS (TLS 1.0+, minimum 128-bit encryption).
- **Maximum message size:** 10 GB (including attachments).

### 13.2 The two function calls

The entire VISI SOAP protocol consists of exactly two function calls:

| Function | Purpose | Input | Output |
|----------|---------|-------|--------|
| **parseMessage** | Send a VISI message | String (XML message + MTOM attachments) | None (transport ACK only) |
| **parseMessageConfirmation** | Confirm receipt | UniqueID (string) + Error list (string) | XML response |

### 13.3 The asynchronous exchange flow

This is the critical protocol flow. It uses **two separate HTTP connections**, not a synchronous request-response. The step numbers below match **Bijlage 8 §3.4** (stap 1–11):

```
Stap 1:  Sending IS composes the VISI message based on the received message
         (or PSB for a new transaction).

Stap 2:  Sending IS extracts sender and receiver URLs FROM THE MESSAGE BODY.
         (Body is the source of truth — "De URL adressen ... worden door het
         versturende IS uit het opgemaakte bericht gehaald.")

Stap 3:  Sending IS hands the message + both URLs to its SOAP server.

Stap 4:  Sending SOAP server builds the SOAP envelope:
         - Header: <sender>, <receiver>, <UniqueID>, <Attachments count>
         - Body: <parseMessage> containing the VISI message + MTOM attachments

Stap 5:  Sending SOAP server sends to Receiving SOAP server (HTTPS).

Stap 6:  Receiving SOAP server forwards the VISI message to Receiving IS.

Stap 7:  Receiving SOAP server returns a standard SOAP transport ACK to Sending
         SOAP server. (This is just "I received bytes" — NOT content validation.)
         — THIS COMPLETES HTTP CONNECTION 1 —

Stap 8:  Receiving IS validates the message (XSD validation, semantic checks).

Stap 9:  Receiving IS builds a parseMessageConfirmation response with error codes.
         Hands it + sender/receiver URLs to its SOAP server.

Stap 10: Receiving SOAP server sends the confirmation to Sending SOAP server
         ON A NEW HTTP CONNECTION (this is the async part — leg 2).

Stap 11: Sending IS receives the confirmation, correlates by UniqueID,
         verifies it matches the sent message, and updates lifecycle status
         (→ Delivered if CODE=0, → Rejected if CODE=1).
```

> **Key distinction:** Stap 7 (transport ACK) and Stap 10 (content confirmation) happen on **different HTTP connections**. The sender transitions to `AwaitingConfirmation` after stap 7, then to `Delivered` or `Rejected` after stap 11.

### 13.4 The SOAP envelope

**Outbound parseMessage:**

The SOAP envelope uses MTOM for efficient attachment transport. Attachments are sent as separate MIME parts referenced via `xop:Include`, not as XML children:

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="..." xmlns:xop="..." xmlns:xmime="...">
  <SOAP-ENV:Header>
    <SOAPServerURL>
      <sender>http://192.168.0.102</sender>
      <receiver>http://192.168.0.138</receiver>
    </SOAPServerURL>
    <UniqueID>
      <ID>UniqueIDonMessageInitiatingSOAPServer_XYZ</ID>
    </UniqueID>
    <Attachments>
      <count>2</count>
    </Attachments>
  </SOAP-ENV:Header>
  <SOAP-ENV:Body>
    <parseMessage>
      <visiXML_MessageSchema>
        ... the VISI message XML ...
        <!-- Attachment metadata appears here as AppendixTemplate elements.
             The actual binary data is in MTOM MIME parts, referenced
             by the AppendixTemplate's id matching the MIME part's Content-ID. -->
      </visiXML_MessageSchema>
    </parseMessage>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>

-- MTOM MIME boundary --
Content-Type: application/octet-stream
Content-ID: <attachment1>
... binary data ...

-- MTOM MIME boundary --
Content-Type: application/octet-stream
Content-ID: <attachment2>
... binary data ...
```

> **Note:** The spec's example (Bijlage 8 §3.4) shows `<Data>` elements as children of `<parseMessage>`, but the actual MTOM wire format places binary attachments as separate MIME parts. The `<Data id="...">` element id matches the MIME part's Content-ID, allowing the receiver to reassemble the message with its attachments.

**Confirmation (no errors):**

```xml
<SOAP-ENV:Envelope>
  <SOAP-ENV:Header>
    <SOAPServerURL>
      <sender>http://192.168.0.102</sender>
      <receiver>http://192.168.0.138</receiver>
    </SOAPServerURL>
    <UniqueID>
      <ID>UniqueIDonMessageInitiatingSOAPServer_XYZ</ID>
    </UniqueID>
  </SOAP-ENV:Header>
  <SOAP-ENV:Body>
    <ERRORS>
      <ERROR CODE="0"></ERROR>
    </ERRORS>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

**Confirmation (with errors):**

```xml
<SOAP-ENV:Body>
  <ERRORS>
    <ERROR CODE="1">Value of simple element 'foo' does not conform to definition</ERROR>
    <ERROR CODE="1">Required attachment missing</ERROR>
  </ERRORS>
</SOAP-ENV:Body>
```

Error codes: **0** = success, **1** = unknown/validation error, **>1** = reserved for future machine-readable codes.

### 13.5 Central Server (optional)

A project may optionally designate a Central Server (`SOAPCentralServerURL` on the ProjectTypeInstance). This is a passive archive that receives a copy of every message after successful bilateral delivery. It is not a router — the bilateral exchange always completes first, then copies are pushed to the Central Server. The Central Server uses the same SOAP protocol (parseMessage + parseMessageConfirmation). If it is unreachable, the bilateral transaction remains valid; the archive simply has a gap.

See `Notes_Timo/central_server.md` for full details.

### 13.6 Receiver-side implementation pattern

The async two-leg model means the receiver must:

1. **ACK immediately** (stap 7) — return a transport-level HTTP 200 to the sender's SOAP server. This must be fast; no validation here.
2. **Validate asynchronously** — queue the inbound message for processing by a background worker. Parse the VISI content, validate against the promoted XSD, run semantic checks.
3. **POST the confirmation** (stap 10) — on a new outbound HTTP connection to the sender's SOAP server URL (from the inbound message's header). Include the UniqueID from the original message and the error codes.

This pattern means the receiver's SOAP endpoint returns quickly while validation happens in the background. The sender tracks a message lifecycle: `Draft` → `AwaitingConfirmation` (after stap 7) → `Delivered` (after stap 11, CODE=0) / `Rejected` (CODE=1) / `ConfirmationTimedOut` (if stap 10 never arrives within a configured threshold — this is a local operational fallback, not in the spec).

### 13.7 Message signing (v1.7)

Version 1.7 adds XAdES-B-LT digital signatures:
- **Messages:** Enveloped signature within the `<parseMessage>` body.
- **Attachments:** Detached signatures (separate file per attachment).
- **Minimum encoding:** SHA256 (SHA384/SHA512 allowed).
- **Certificate:** Must be from a trusted issuer (not a test certificate).
- **Attachment checksums:** The `checksum` field on AppendixTemplate stores the SHA256+ hash.

---

## 14. Version Differences (v1.3 → v1.7)

| Feature | v1.3 | v1.4 | v1.6 | v1.7 |
|---------|------|------|------|------|
| Namespace | `20110819` | `20140331` | `20160331` | `20220930` |
| Max attachment size | 120 MB | 120 MB | 10 GB | 10 GB |
| MITTCondition | — | Added | Unchanged | Unchanged |
| HTTPS mandatory | No | Yes | Yes | Yes |
| ComplexElementType minOccurs/maxOccurs | — | — | Added | Unchanged |
| MessageType appendixMandatory | — | — | Added | Unchanged |
| TransactionType.subTransactions | Present | Present | Present | **Removed** |
| TransactionType.result | Present | Present | Present | **Removed** |
| MITT.requiredNotify/received/send | Present | Present | Present | **Removed** |
| SimpleElementType.interfaceType/valueList | Present | Present | Present | **Removed** |
| startDate/endDate on most types | Present | Present | Present | **Removed** |
| AppendixTemplate.checksum | — | — | — | Added |
| Message signing (XAdES) | — | — | — | Added |
| Non-ASCII support | — | — | — | Mandated |

**Version detection:** Parse the `xmlns` namespace from the root element. The namespace URL uniquely identifies the version.

**Cross-version note:** The v1.7 removal of `subTransactions` from TransactionType is not lossy — the parent-child relationship was already expressed through the MITT graph (via cross-transaction `previous` references + `firstMessage=true`). The `subTransactions` field was metadata/documentation, not the actual runtime mechanism.

---

## Appendix A: DEMO Transaction Pattern

VISI is based on DEMO (Design & Engineering Methodology for Organizations). DEMO models cooperation as transactions between roles:

- Every transaction has an **initiator** (who wants something done) and an **executor** (who does it).
- Communication follows a pattern of **intentions**: Request, Promise, State, Accept (or Reject, Revoke, etc.).
- VISI maps these to **TransactionPhaseTypes**: `start`, `verzocht` (requested), `beloofdExecutie` (promised execution), `meldingGereed` (declared done), `aanvaardEinde` (accepted/end), `wijzigingHold` (modification hold).

In VISI, unlike pure DEMO, messages alternate strictly between initiator and executor. Some DEMO transaction states that would involve the same party sending consecutive messages are collapsed.

For full DEMO background: see `docs/visi-1.7-spec/DemoTransactionpattern.md`.

---

## Appendix B: Design Rationale: Non-Obvious Decisions

### Why are SOAP URLs in both the header and the message body?

Three reasons:

1. **SOAP servers cannot parse VISI content** (Bijlage 8 §3 randvoorwaarde 1). The SOAP server needs the URLs to know where to send the message, but it can't extract them from the body. So the IS extracts the URLs from the body and provides them as header metadata.

2. **The body is the source of truth** (Bijlage 8 §3.4 stap 2): *"De URL adressen van het versturende en het ontvangende IS worden door het versturende IS uit het opgemaakte bericht gehaald."* Direction is always body → header, never the reverse.

3. **Self-containment** (Bijlage 8 §3 randvoorwaarde 3): All configuration must be "gevat in VISI berichten" — contained in VISI messages. This allows the receiver to discover reply URLs without external lookups.

The duplication is a consequence of the layered architecture (VISI-unaware SOAP transport under semantically rich VISI content), not redundancy that can be eliminated.

### Why carry the full transitive closure in every message?

Because the EXPRESS model (_5.exp) requires it. The non-OPTIONAL fields of `MessageTemplate` → `TransactionTemplate` → `PersonInRole` → `OrganisationTemplate`/`PersonTemplate`/`RoleTemplate` create a mandatory chain. This is not a convention — implementations cannot omit it.

The design goal is **self-traceability**: any single message, without external context, can be fully placed in its transaction. This supports:
- Dispute resolution (each message is a complete record)
- Archival (messages are independently interpretable)
- Implementation simplicity (no external state queries needed)

### Why don't continuation messages re-read the PSB?

Because transactions are frozen on their original state. Bijlage 8 §3.4 stap 1 specifies that continuation messages are composed from the received message, not the PSB. This means:
- A PSB update mid-transaction doesn't affect ongoing transactions.
- The conversation carries itself — once started, it's self-sustaining.
- Only **new** transactions use the updated PSB.

### Why is the promoted XSD a choice with unbounded maxOccurs?

The `<xs:choice maxOccurs="unbounded">` at the root is the XML serialization of the EXPRESS embedded-object model. In EXPRESS, objects are embedded (owned by their parent). In XML, this is represented by floating them to the root level and using `<XRef idref="..."/>` references. The XSD must allow any number of entities in any order at root level because different messages embed different numbers of entities.

### Why does the PSB share the same schema as regular messages?

Schema-technically, there is no "PSB type" versus "message type." Both are `visiXML_MessageSchema` instances that fill different subsets of the same `<xs:choice>`. The PSB fills configuration entities (persons, orgs, roles, project). Messages fill those same entities plus messaging entities (MessageTemplate, TransactionTemplate, MITT). The name "project-specific message" describes the function, not a distinct file format.

---

## Appendix C: Reference Documents

### Normative specifications

| Document | Description | Path |
|----------|-------------|------|
| VISI 1.7 Summary | Comprehensive reference of all entities, rules, and v1.6→1.7 changes | `docs/visi-1.7-spec/Summary.md` |
| VISI 1.7 Exchange | SOAP protocol, message signing, configuration distribution | `docs/visi-1.7-spec/Exchange.md` |
| VISI 1.7 ProjectDesign | All framework entity types and their attributes | `docs/visi-1.7-spec/ProjectDesign.md` |
| VISI 1.7 ProjectExecution | Message composition, attachments, field conditions | `docs/visi-1.7-spec/ProjectExecution.md` |
| VISI 1.7 MessageTemplates | All message-level (Part II) entity types | `docs/visi-1.7-spec/MessageTemplates.md` |
| VISI 1.7 Technical | Layer terminology, optional field reduction, validation rules | `docs/visi-1.7-spec/Technical.md` |
| VISI 1.7 Example Implementation | Complete worked example with XML | `docs/visi-1.7-spec/Example Implementation.md` |
| VISI 1.7 Delegation | Successor chains and temporary delegation | `docs/visi-1.7-spec/Delegation.md` |
| VISI 1.7 DEMO Pattern | DEMO transaction pattern mapping | `docs/visi-1.7-spec/DemoTransactionpattern.md` |
| VISI 1.6 SOAP (Bijlage 8) | Detailed SOAP protocol steps (Dutch) | `docs/visi-1.6-spec/soap.md` |
| VISI 1.6 Bijlage 3 | EXPRESS Part II data model (Dutch) | Referenced in `Notes_Timo/message_composition.md` |

### EXPRESS schemas

| File | Version | Purpose |
|------|---------|---------|
| `_2.exp` (v1.7) | `20220930` | Framework data model — all entity type definitions |
| `_5.exp` (v1.7) | `20220930` | Message instance model — all template/instance definitions |
| `_2.exp` (v1.6) | `20160331` | Framework data model (includes startDate/endDate, subTransactions) |
| `_5.exp` (v1.6) | `20160331` | Message instance model (includes startDate/endDate) |

### Example data

| File | Description |
|------|-------------|
| `testproject/topkoks/_7.xml` | TopKoks (pizzeria) framework |
| `testproject/topkoks/10.xsd` | Promoted XSD for TopKoks |
| `testproject/topkoks/project_specifiek_bericht_A.xml` | PSB for TopKoks |
| `testproject/topkoks/voorbeeld_bericht.xml` | Example message in TopKoks |
| `testproject/praktijkvoorbeelden/` | Real-world practice examples |

### Existing notes

| File | Topic |
|------|-------|
| `Notes_Timo/promotor.md` | How the Promotor DLL works, file pipeline |
| `Notes_Timo/message_composition.md` | Message self-containment, PSB slicing, URL duplication rationale |
| `Notes_Timo/central_server.md` | SOAP Central Server architecture and reliability |
| `Notes_Timo/version_overview_and_subTransactions.md` | Version differences, MITT graph, subTransaction removal |
