High-Level Overview

  This repo is the official management and publication environment for the VISI open standard — a Dutch open standard for structured communication in construction projects. It's managed by CROW/DigiGO under CC-BY-4.0.

  Contents: Mostly documentation (PDFs, DOCX, Markdown) and XML/XSD schema definitions. The only real code is a legacy C++ promotor DLL in 3_visi_promotor_software/. The docs/ folder serves a GitHub Pages site with ReSpec documentation for versions 1.6, 1.7, and
   1.8.

  Core concept: VISI defines frameworks (raamwerken) that specify roles, transactions, and messages for project communication. The _3.xsd is the meta-schema that validates framework XML files.

  ---
  Version-by-Version Changes (1.3 → 1.7)

  v1.3 (namespace: 20110819) — baseline

  - 14 entity types, SOAP communication, max 120MB attachments
  - Introduced UUIDs for transactions, meta-framework, successor mechanism
  - Fields like startDate, endDate, category present on most types
  - subTransactions and result on TransactionType

  v1.4 (namespace: 20140331)

  - Added MessageInTransactionTypeCondition — new entity enabling sendAfter/sendBefore message ordering constraints
  - HTTPS/TLS mandatory for all SOAP communication
  - Otherwise structurally nearly identical to 1.3

  v1.6 (namespace: 20160331)

  - ComplexElementType gains minOccurs/maxOccurs — table row count control
  - MessageType gains appendixMandatory boolean
  - ElementCondition.complexElement → complexElements (singular→plural, the only true structural rename)
  - Attachment max raised to 10GB
  - Clarified ElementCondition priority for tables

  v1.7 (namespace: 20220930)

  - Major cleanup — removed deprecated fields from most types:
    - Stripped startDate, endDate, category from ~10 entity types
    - Stripped requiredNotify, received, send from MITT
    - Stripped interfaceType, valueList from SimpleElementType
    - Stripped result, subTransactions from TransactionType
  - Non-ASCII character support mandated
  - Documentation translated to English

  ---
  Architect Recommendations (Multi-Version Software)

  The architect agent completed a full analysis. Key recommendations:

  1. Version detection: Parse xmlns namespace from root element — single string comparison, completely reliable
  2. Data model: Superset union type — One set of entity types where every field from every version is optional. Tag each framework with its source version. The schemas overlap ~85-90%, making per-version models wasteful.
  3. Validation: XSD + custom — First validate XML against the bundled version-specific _3.xsd, then run custom semantic checks (referential integrity, MITT graph consistency, ElementCondition validity)
  4. Cross-version exchange: Parties must agree on a shared framework. Downgrading (e.g., 1.7→1.6) means re-adding removed optional fields; upgrading means stripping them. The subTransactions removal in 1.7 is potentially lossy.
  5. Tech stack: TypeScript or C# recommended. DOM parsing (frameworks are small). Bundle all 5 XSD files as embedded resources keyed by namespace URL.
  6. Implementation sequence: Version detector → Generic parser → XSD validator → Semantic validator → Version converter → Promotor replacement

  
  
  
  Correction: subTransactions was indeed removed, but it was already rarely used
  
    You're right to question this. Let me be precise about what I found:
  
    What actually happened with subTransactions
  
    v1.3/v1.4 _2.exp (Part I / framework schema):
    ENTITY TransactionType;
      ...
      result : OPTIONAL STRING;
      subTransactions : OPTIONAL SET [1:?] OF TransactionType;
      initiator : RoleType;
      executor : RoleType;
      ...
    END_ENTITY;
  
    v1.6 _2.exp: Same — subTransactions and result still present.
  
    v1.7 _2.exp: Both subTransactions and result are gone. Confirmed in both the EXPRESS model and the XSD (_3.xsd).
  
    v1.8 _2.exp: Still gone.
  
    So yes, subTransactions on TransactionType was truly removed in v1.7. However — and this is the important nuance — even in the v1.6 demo framework (_7.xml), none of the four TransactionTypes actually use subTransactions. The relationship between transactions
    was already being expressed elsewhere.
  
    How transactions relate to each other in ALL versions (including 1.7)
  
    The parent-child transaction relationship is not primarily expressed via subTransactions. It is expressed through the MessageInTransactionType (MITT) graph. Here's how:
  
    1. Each MITT references exactly one TransactionType via <transaction>. This is the transaction that MITT belongs to.
    2. openSecondaryTransactionsAllowed (boolean on MITT, present in v1.3–v1.7, removed in v1.8) — When true on a MITT, it means that when this message is sent, the software should allow the user to open new ("secondary") transactions. This is how a parent
    transaction spawns child transactions.
    3. previous (on MITT) — Links to predecessor MITTs. This is how the message flow within a single transaction is defined as a directed graph.
    4. firstMessage (boolean on MITT) — Marks the entry point of a transaction.
  
    So the actual mechanism for message exchange within a transaction in v1.7 is:
  
    TransactionType (defines initiator role + executor role)
      └── MessageInTransactionType (many, linked by `previous`)
            ├── message → MessageType (what data to carry)
            ├── transaction → TransactionType (which transaction this belongs to)
            ├── initiatorToExecutor → boolean (direction of this message)
            ├── firstMessage → boolean (is this the starting message?)
            ├── previous → [MITT refs] (what must come before this message)
            ├── openSecondaryTransactionsAllowed → boolean (can start sub-transactions from here)
            ├── transactionPhase → TransactionPhaseType (optional phase label)
            ├── conditions → [MITTCondition refs] (sendAfter/sendBefore constraints)
            └── group → GroupType (for appendix grouping)
  
    The key insight: subTransactions was redundant
  
    The subTransactions field on TransactionType was essentially metadata/documentation — it declared "these transaction types may appear as children." But the actual runtime behavior was always controlled by openSecondaryTransactionsAllowed on individual MITTs.
    When a MITT with openSecondaryTransactionsAllowed=true is reached, the software looks at which other TransactionTypes exist in the framework and allows the user to start them.
  
    The v1.7 removal just acknowledged this: the sub-transaction relationship is implicit from the MITT graph, not from an explicit subTransactions list.
  
    v1.8 changes the sub-transaction story further
  
    In v1.8, openSecondaryTransactionsAllowed is itself removed from MITT. Instead, v1.8 adds:
    - revokeAllowed (boolean) — can this message be revoked?
    - revokePattern (ref to TransactionType) — which transaction pattern to use for revocation
  
    This suggests v1.8 is moving toward a model where sub-transactions are more explicitly tied to specific patterns rather than a generic "allow any secondary transaction" flag.
  
    Summary of my correction
  
    My earlier statement was factually correct (subTransactions was removed from TransactionType in v1.7) but misleading about its impact. The removal was not lossy in practice because:
    - The actual transaction flow was always defined by the MITT graph
    - openSecondaryTransactionsAllowed on MITT was the real mechanism for sub-transactions
    - The subTransactions field was rarely populated even when available
  




 MITT (MessageInTransactionType) — Complete Explanation

  Where it's defined

  MITT exists in two places:

  Part I — Framework schema (_2.exp / _3.xsd): Defines the type — which messages can be sent, in what order, within which transaction. This is the design-time definition that framework creators author.

  Part II — Message schema (_5.exp): Defines the instance — MessageInTransactionTemplate is the runtime record of a message actually sent. It's much simpler: just identification, dateSend, dateRead, state, dateLaMu, userLaMu.

  The complexity is entirely in Part I. Let me walk through the v1.7 structure.

  MITT structure (v1.7 _2.exp)

  ENTITY MessageInTransactionType;
    dateLaMu                          : OPTIONAL DATETIME;
    userLaMu                          : OPTIONAL STRING;
    state                             : OPTIONAL STRING;
    initiatorToExecutor               : OPTIONAL BOOLEAN;
    openSecondaryTransactionsAllowed  : OPTIONAL BOOLEAN;
    firstMessage                      : OPTIONAL BOOLEAN;

    message          : MessageType;                                    -- REQUIRED
    previous         : OPTIONAL SET [0:?] OF MessageInTransactionType; -- predecessor MITTs
    transaction      : TransactionType;                                -- REQUIRED
    transactionPhase : OPTIONAL TransactionPhaseType;
    group            : OPTIONAL GroupType;
    appendixTypes    : OPTIONAL SET [1:?] OF AppendixType;
    conditions       : OPTIONAL SET [1:?] OF MessageInTransactionTypeCondition;
  END_ENTITY;

  What each field means for software

  ┌──────────────────────────────────┬───────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │              Field               │                          Purpose                          │                                            Software implication                                             │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ message                          │ Which MessageType is sent at this step                    │ Determines the form/data structure the user fills in                                                        │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ transaction                      │ Which TransactionType this MITT belongs to                │ Used to group MITTs into transaction flows                                                                  │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ initiatorToExecutor              │ Direction of this message                                 │ true = initiator sends to executor; false = executor sends to initiator                                     │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ firstMessage                     │ Is this the entry point?                                  │ true = this MITT can start a new transaction instance                                                       │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ previous                         │ Which MITTs must precede this one                         │ Defines the directed graph — a MITT is only available when at least one of its previous MITTs has been sent │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ openSecondaryTransactionsAllowed │ Can sub-transactions be started here?                     │ When true and this message has been sent, show the user the option to start new transactions                │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ transactionPhase                 │ Labels the state (e.g., "start", "requested", "accepted") │ For display/tracking purposes                                                                               │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ conditions                       │ Extra constraints via MITTCondition                       │ Adds sendAfter/sendBefore rules (see below)                                                                 │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ group                            │ Appendix grouping                                         │ Determines which appendix group this message's attachments belong to                                        │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ appendixTypes                    │ Which appendix types are allowed                          │ Restricts which attachment types the user can add                                                           │
  └──────────────────────────────────┴───────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  How the MITT graph works — traced through the Pizzeria example

  The v1.7 test framework models a restaurant with 4 transactions across 4 roles (klant/customer, ober/waiter, kok/chef, keukenhulp/kitchen helper). Here's transaction t1_OpnameBestelling (taking an order):

                            t1_OpnameBestelling (ober→klant)
                            ================================

    [MITT1] "Wil u de kaart zien?"     ober→klant     firstMessage=true    phase=start
       │
       ├──► [MITT2a] "Nee"             klant→ober                          phase=aanvaardEinde
       ├──► [MITT2b] "Nee met comment" klant→ober                          phase=aanvaardEinde
       └──► [MITT3]  "Ja"              klant→ober                          phase=verzocht
                │
                └──► [MITT4] "Aanbieding menukaart"  ober→klant            phase=beloofdExecutie
                        │
                        ├──► [MITT5] "Vraag"         klant→ober            phase=wijzigingHold
                        │       │
                        │       └──► [MITT6] "Antwoord" ober→klant  openSecondary=true
                        │               │                           phase=beloofdExecutie
                        │               └──► (loops back to MITT5 or MITT7)
                        │
                        └──► [MITT7] "Plaatsing bestelling"  klant→ober    phase=meldingGereed
                                │
                                └──► [MITT12] firstMessage=true  ──► starts t3_OpdrachtKok!

  Key observations:

  1. Branching: MITT1 has three possible continuations (MITT2a, MITT2b, MITT3). Software must present all valid options to the active user.
  2. Multiple predecessors: MITT5 has previous = [MITT4, MITT6, MITT8]. This means MITT5 is available when any of those has been sent. It's an OR, not an AND.
  3. Cross-transaction links: MITT12 belongs to t3_OpdrachtKok but has previous = [MITT7] which belongs to t1_OpnameBestelling. This is how transactions chain — a MITT with firstMessage=true can have a previous pointing to a MITT in another transaction. This is
  the mechanism that replaced subTransactions.
  4. openSecondaryTransactionsAllowed=true: On MITT6 and MITT13/14 — after the waiter answers a question or the chef accepts/rejects, secondary transactions can be spawned.

  Cross-transaction MITT references — the key mechanism

  This is the critical part software must handle. Looking at the Sendafter test case in v1.7:

  TR_Sendafter_andConditie_hoofdtransactie  (RL_A → RL_B)
      └── MITT sends message, then...
          ├── TR_Sendafter_andConditie_EersteSubtransactie  (RL_B → RL_C)
          │       └── MITTs with firstMessage=true, previous pointing to parent transaction's MITT
          └── TR_Sendafter_andConditie_TweedeSubtransactie  (RL_B → RL_D)
                  └── MITTs with conditions using sendAfter referencing first subtransaction's MITTs

  The MITTCondition (MC_13846, MC_13847) enforces ordering across transactions: "only offer the second sub-transaction after the first one has been sent."

  How MITTCondition works

  <MessageInTransactionTypeCondition id="MC_MaarEenBestellingToegestaan">
      <sendBefore>
          <MessageInTransactionTypeRef idref="BerichtInTransactie12"/>
      </sendBefore>
  </MessageInTransactionTypeCondition>

  - sendBefore: This MITT may only be sent as long as the referenced MITTs have NOT yet been sent. (Blocks after they're sent.)
  - sendAfter: This MITT may only be sent after the referenced MITTs have been sent. (Requires them first.)

  Both can reference MITTs in other transactions — this is how cross-transaction ordering is enforced.

  What software must implement

  1. Build the MITT graph per transaction at framework load time:
  For each TransactionType:
      Collect all MITTs where mitt.transaction == this TransactionType
      Build a directed graph: edges from mitt.previous → mitt
      Identify entry points: MITTs where firstMessage == true

  2. Build the cross-transaction graph:
  For each MITT with firstMessage == true AND previous != empty:
      The previous refs point to MITTs in OTHER transactions
      This defines the "spawn" relationship:
          parent transaction's MITT → child transaction starts

  3. At runtime, determine available MITTs:
  def get_available_mitts(transaction_instance, all_sent_messages):
      available = []
      for mitt in transaction_instance.type.mitts:
          # Check previous: at least one previous must have been sent
          if mitt.previous:
              if not any(is_sent(prev, all_sent_messages) for prev in mitt.previous):
                  continue

          # Check conditions
          if mitt.conditions:
              for condition in mitt.conditions:
                  if condition.sendAfter:
                      # ALL sendAfter refs must have been sent
                      if not all(is_sent(ref, all_sent_messages) for ref in condition.sendAfter):
                          skip mitt

                  if condition.sendBefore:
                      # ALL sendBefore refs must NOT have been sent
                      if any(is_sent(ref, all_sent_messages) for ref in condition.sendBefore):
                          skip mitt

          # Check direction: only show to the right user
          if mitt.initiatorToExecutor:
              only show to user with initiator role
          else:
              only show to user with executor role

          available.append(mitt)
      return available

  4. Handle openSecondaryTransactionsAllowed:
  When a message is sent for a MITT with openSecondaryTransactionsAllowed == true:
      Find all MITTs in the framework where:
          - firstMessage == true
          - previous contains any MITT that has been sent in this or related transactions
      Offer those as new transactions the user can start

  5. The previous semantics are crucial:
  - Multiple items in previous = OR (any one predecessor suffices)
  - sendAfter with multiple items = AND (all must be sent)
  - sendBefore with multiple items = AND (none may be sent)

  This OR-vs-AND distinction between previous and conditions is what makes the system expressive enough to model complex flows without needing explicit sub-transaction declarations.
