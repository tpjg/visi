SOAP Central Server

  What it is

  A passive archive server that receives a copy of every inter-organizational message exchanged in a VISI project. It does not route, process, or understand VISI messages — it just stores them and acknowledges receipt.

  How it differs from regular SOAP Servers

  ┌───────────────────────┬───────────────────────────────────────────────────┬─────────────────────────────────────────────┐
  │                       │                    SOAP Server                    │             SOAP Central Server             │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ One per               │ Organisation                                      │ Project (0 to many)                         │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ Purpose               │ Send/receive messages for an organisation's users │ Store copies of all bilateral communication │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ VISI awareness        │ None (by design)                                  │ None (by design)                            │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ Configured in         │ SOAPServerURL on OrganisationType                 │ SOAPCentralServerURL on ProjectType         │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ Cardinality           │ Exactly 1 per organisation                        │ 0..n per project                            │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ Actively participates │ Yes — sends and receives                          │ No — only receives copies                   │
  ├───────────────────────┼───────────────────────────────────────────────────┼─────────────────────────────────────────────┤
  │ Mandatory             │ Yes                                               │ No                                          │
  └───────────────────────┴───────────────────────────────────────────────────┴─────────────────────────────────────────────┘

  Its role in the message flow (Bijlage 8, section 3.4)

  The Central Server is woven into the 11-step message exchange scenario:

  1. Sender's IS composes the VISI message
  2. Sender's IS looks up sender URL, receiver URL, and Central Server URL(s) from the project-specific message
  3. Sender's SOAP Server builds the SOAP envelope (with sender/receiver/central URLs in the header)
  4. If a Central Server exists: message goes to the Central Server first, then to the receiver. If no Central Server: message goes directly to the receiver.
  5. After successful delivery to the receiver (confirmed via SOAP acknowledgment), a copy is also sent to all Central Server(s)
  6. The receiver's confirmation message is likewise copied to all Central Server(s)

  Key rule from p.15: "In stap 7 en stap 10 is een bericht altijd eerst naar de andere partij gestuurd. Pas als via het standaard SOAP protocol een goede reactie is ontvangen wordt een kopie naar de SOAP Central Server(s) gestuurd." — The actual bilateral
  exchange always takes priority; the Central Server only gets copies after successful delivery.

  What it does NOT do

  - It does not route messages between parties
  - It does not parse or validate VISI content
  - It does not have any VISI-specific logic
  - It does not handle intra-organisational messages (those never leave the local system)
  - What happens with stored messages, who has access, and how they're secured is explicitly out of scope of the protocol (Bijlage 8, p.5)

  Protocol: identical to bilateral SOAP traffic

  There is no separate "archive protocol" — the Central Server is just another SOAP endpoint, addressed via the same MTOM SOAP envelope used for bilateral exchange:
  - Only one protocol is defined project-wide (`SOAPProtocol` = `MTOM`, in the projectspecifieke bericht). It applies to all SOAP traffic regardless of destination.
  - Only two function calls exist in the whole protocol (Bijlage 8 §8): `parseMessage` and `parseMessageConfirmation`. The same calls are used for bilateral delivery and for the copy to the Central Server.
  - The message pushed to the Central Server is an "exacte kopie van gecommuniceerd bericht" (Bijlage 8 §2) — byte-identical envelope, same `<SOAPServerURL>` / `<SOAPCentralServerURL>` / `<UniqueID>` header slots.
  - Both regular and Central SOAP servers are explicitly required to be unable to parse VISI content ("niet in staat VISI berichten te parsen", §3) — they are interchangeable from a protocol-capability standpoint.

  Reliability: best-effort, no retries

  Bijlage 8 defines a synchronous per-message ack but no recovery mechanism. In practice the Central Server is best-effort archiving:

  - Per-message confirmation exists: the Central Server must "[vermelden] aan de verzendende SOAP server dat het bericht in goede orde ontvangen is" (§2). Acknowledgement uses the same SOAP `parseMessageConfirmation` / `ERROR CODE=0` envelope as bilateral traffic.
  - No retries — anywhere. §3.4 bullet (p.15): "Indien er niet ('op tijd') gereageerd wordt op een bericht wordt niet nogmaals hetzelfde bericht verstuurd. We blijven wachten op het antwoord of vinden een oplossing buiten VISI om." This applies to all SOAP delivery, including copies to the Central Server.
  - No failure handling for the archive path. The ordering rule (§3.4, steps 7/10) is that the bilateral exchange always happens first; the Central Server copy is only pushed after successful bilateral delivery. If that subsequent push fails, the bilateral transaction is already complete — the protocol specifies no queue, no rollback, no retry, no "message missing from archive" reconciliation query.
  - No durability / integrity / access guarantees. §0 (p.5) explicitly scopes this out: "Wat er vervolgens met deze berichten gedaan wordt, welke partijen toegang tot welke berichten hebben en hoe de beveiliging en opslag van deze berichten is geregeld valt buiten dit protocol."

  Consequence: if the Central Server is unreachable at the moment a copy would be pushed, that message is simply absent from the archive. The bilateral transaction remains fully valid, and any remediation must happen outside VISI. This is a deliberate trade-off from §2: the architecture requires bilateral communication to work "zonder tussenkomst van andere servers" — reliability of the archive was sacrificed to keep the bilateral path independent and non-blocking.

  Configuration

  The Central Server URL is declared in the project-specific message (not in the raamwerk), under the SOAPCentralServerURL SimpleElementType on the ProjectType:

  <AnderWillekeurigComplexElement id="ProjectGegevens">
    <SOAPCentralServerURL>http://192.168.0.1/visi.wsdl</SOAPCentralServerURL>
  </AnderWillekeurigComplexElement>

  If this field is empty, the system simply operates without a Central Server — making it fully optional.

  Purpose / why it exists

  The Hoofddocument (p.5) states: "Het doel van de SOAP Central Server is het opslaan van alle berichten (m.u.v. berichten binnen 1 organisatie) die binnen een project plaatsvinden."

  In essence, it's a project-level audit trail / escrow mechanism — a neutral third-party server that collects a complete record of all formal cross-organisational communication. This supports:
  - Dispute resolution (one authoritative copy of all exchanges)
  - Compliance / archiving (Dutch Archiefwet, referenced in Bijlage 10)
  - Transparency for project owners who want oversight without being a party to every transaction

  It's architecturally similar to a BCC on every email — except it's a dedicated server that both parties agree to as part of the project setup.
