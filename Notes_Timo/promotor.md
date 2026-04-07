i  The Promotor: What It Does

The Promotor is a schema compiler for the VISI standard. It's a C++ Windows DLL (with GUI and CLI wrappers) that transforms abstract definitions into a concrete XML Schema that can validate VISI messages.

Input (3 files)

┌─────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────┐
│ File suffix │                                                     What it is                                                     │                       Example                       │
├─────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
│ _2.exp      │ EXPRESS schema — the VISI data model itself (entity definitions like TransactionType, MessageType, RoleType, etc.) │ Defines what types exist and their attributes       │
├─────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
│ _5.exp      │ EXPRESS instance schema — defines how instances/templates of those entities look                                   │ Defines how instances are structured                │
├─────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
│ _7.xml      │ The raamwerk (framework) — an XML file containing the actual project-specific roles, transactions, messages, etc.  │ The concrete communication agreements for a project │
└─────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────┘

Output

┌──────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────┐
│             File suffix              │                               What it is                                │
├──────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ _9.xsd (via Promote__2__5__7into__9) │ An XSD schema that can validate VISI messages exchanged in that project │
├──────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ _3.xsd (via Convert__2into__3)       │ A generic XSD from just the EXPRESS model (no raamwerk)                 │
├──────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ _10.xsd (via Convert__9into_10)      │ An XSD for validating the raamwerk XML itself                           │
└──────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────┘

The core operation: "Promotion"

The key function Promote__2__5__7into__9() does this:

1. Parses the EXPRESS schema (_2.exp) — understands entity types, attributes, inheritance, optionality, base types (BOOLEAN, STRING, INTEGER, DATE, etc.)
2. Parses the EXPRESS instance schema (_5.exp) — understands template vs. instance vs. virtual element patterns
3. Parses the raamwerk XML (_7.xml) — reads the concrete project definitions
4. Matches each entity in the raamwerk against its schema definition, resolving templates and instances
5. Generates a project-specific XSD (_9) that includes only the types actually used in that raamwerk, with proper nesting, restrictions, and namespace

Why it exists

A VISI raamwerk defines which roles, transactions, and messages exist for a project, but it's just data. To actually validate that messages exchanged between SOAP servers conform to that raamwerk, you need an XSD schema. The Promotor generates that XSD
automatically.

Think of it as: raamwerk (what we agreed to communicate) → XSD (machine-enforceable rules for those communications)

The name "Promotor" comes from this act of "promoting" an abstract framework definition into a concrete, enforceable schema.




The file numbering system

  The VISI definition files form a pipeline, each number representing a transformation stage:

  ┌─────────┬────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │  File   │          Name          │                                                       What it contains                                                        │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _2.exp  │ EXPRESS schema Part I  │ The VISI data model — defines all entity types (ProjectType, RoleType, MessageType, etc.) and their attributes                │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _3.xsd  │ XSD from _2            │ XSD to validate raamwerken — auto-generated from _2.exp                                                                       │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _5.exp  │ EXPRESS schema Part II │ The message instance model — defines how instances look at runtime (MessageTemplate, TransactionTemplate, PersonInRole, etc.) │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _7.xml  │ Raamwerk               │ A specific framework — concrete definitions of roles, transactions, messages for a project                                    │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _9.xsd  │ Promoted schema        │ XSD to validate VISI messages — generated from _2 + _5 + _7 combined                                                          │
  ├─────────┼────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ _10.xsd │ ???                    │ See below                                                                                                                     │
  └─────────┴────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  _3.xsd — "Can I validate a raamwerk?"

  Generated by: Convert__2into__3() (the exp2xsd tool)

  Input: _2.exp only

  This is straightforward: take the EXPRESS entity definitions and convert them to XSD. Looking at the actual file:

  - Root element: <visiXML_VISI_Systematics>
  - Contains generic VISI type names: AppendixType, ComplexElementType, MessageType, RoleType, TransactionType, etc.
  - Namespace: http://www.visi.nl/schemas/20160331

  Purpose: Validate that a raamwerk XML (_7.xml) is structurally correct according to the VISI data model. This is version-specific but project-independent.

  _9.xsd — "Can I validate a message in this project?"

  Generated by: Promote__2__5__7into__9() (the Promotor's core function)

  Input: _2.exp + _5.exp + _7.xml (all three)

  This is the "promotion" step. Looking at the 10.xsd file (which is a _9-type output for the test framework):

  - Root element: <visiXML_MessageSchema>
  - Contains project-specific names from the raamwerk: msgAanbiedingMenuKaart, msgPlaatsingBestelling, CeBestelling, kok, ober, klant, t1_OpnameBestelling, etc.
  - Namespace: http://www.visi.nl/schemas/20160331/TestFramework (raamwerk-specific!)

  Purpose: Validate that actual VISI messages exchanged via SOAP conform to this specific raamwerk. The entity types from _2.exp are merged with the instance structure from _5.exp and the concrete ids from _7.xml to produce an XSD where every element name is a
  real id from the raamwerk.

  _10.xsd — "A promoted schema with a specific namespace"

  Generated by: Convert__9into_10() followed by Get_10()

  This is essentially the same as _9, but with an explicit namespace and URI applied. Looking at the DLL API:

  int Get__3(char ** __3, char ** rInfo, char * nameSpace, char * URI);
  int Get_10(char ** _10, char ** rInfo, char * nameSpace, char * URI);

  Both Get__3 and Get_10 take a nameSpace and URI parameter. The _10 is the final, publishable form of the promoted schema — the _9 with the correct project-specific namespace stamped in (like http://www.visi.nl/schemas/20160331/TestFramework).

  In practice, the file called 10.xsd in the repo IS the promoted schema for the test raamwerk. It's what VISI software uses at runtime to validate incoming SOAP messages against.

  The full pipeline visually

  _2.exp (VISI data model)
    │
    ├──→ exp2xsd ──→ _3.xsd  (validates raamwerken)
    │
    │   _5.exp (instance model)
    │     │
    │     │   _7.xml (a specific raamwerk)
    │     │     │
    └─────┴─────┴──→ Promotor ──→ _9/_10.xsd  (validates messages for THAT raamwerk)

✻ Cogitated for 1m 10s

❯ What is the benefit / how does the namespacing for 10.xsd come into play compared to _9.xsd?

⏺ The key is in the DLL API and the changelog. Let me re-read the relevant code.

⏺ Read 1 file (ctrl+o to expand)

⏺ Looking at the code, here's the honest answer:

  There is no functional difference between _9 and _10. They're the same thing.

  The code reveals this clearly:

  - Get__3() and Get_10() are identical — both call buildXSD(&XSD, &info, nameSpace, URI) with the same parameters
  - Convert__2into__3() and Convert__9into_10() are identical — both call setGlobalSchema() to prepare the internal state
  - The only difference is which internal state they operate on (set SET__2 vs SET__9)

  The _9 is the in-memory result of promotion. It doesn't have a namespace/URI baked in yet — it's the raw promoted schema sitting inside the DLL's internal data structures.

  The _10 is what you get when you call Get_10(nameSpace, URI) — the same promoted schema serialized to XSD text with the namespace and URI written into the targetNamespace attribute.

  So the workflow is:

  1. Set__2(file)          → parse _2.exp into memory
  2. Set__5(file)          → parse _5.exp into memory
  3. Set__7(file)          → parse _7.xml into memory
  4. Promote__2__5__7into__9()  → combine all three → promoted schema in memory (_9)
  5. Get_10(&output, nameSpace, URI)  → serialize _9 to XSD string with namespace → _10

  The namespace matters because each raamwerk gets its own namespace (e.g. http://www.visi.nl/schemas/20160331/TestFramework). This is how VISI guarantees traceability after raamwerk changes (Hoofddocument, TC001): messages validated against raamwerk version A
  won't accidentally validate against raamwerk version B, because their XSD targetNamespace values differ.

  The _9 → _10 distinction is really just an API design artifact: separating the "compute" step from the "serialize with parameters" step, so the caller can stamp in whatever namespace applies to their project.

❯ Is the namespace itself taken from _7.xml?

✻ Cooked for 3m 36s

❯ Is the namespace itself taken from _7.xml or is it supplied in a different way?

⏺ There it is, lines 851-870. It's a fallback mechanism:

  1. The caller can supply a URI parameter explicitly via Get_10(output, nameSpace, URI)
  2. If the URI is null or empty (URI == 0 || URI[0] == 0), the code walks through the parsed _7.xml elements looking for an attribute called "namespace" and uses its value as the URI

  This matches the _2.exp definition where ProjectType has namespace : STRING as its first (non-optional!) attribute, and the changelog entry for v0.6: "In geval de DLL geen URI (danwel een lege URI) binnenkrijgt zal de functie de URI proberen te halen uit het
  raamwerk (een instance van 'namespace')."

  So looking at the test raamwerk _7.xml, the ProjectType would have something like <namespace>http://www.visi.nl/schemas/20160331/TestFramework</namespace>, and that's where the targetNamespace in the output XSD comes from when the caller doesn't specify one.

  In short: the URI can be supplied by the caller, but if omitted, yes — it's extracted from the namespace field of the ProjectType in _7.xml.
