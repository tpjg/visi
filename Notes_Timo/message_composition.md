# VISI bericht‑opbouw: Types, Templates en de PSB‑slice

Hoe een VISI‑bericht écht is opgebouwd, waar elk stuk vandaan komt, en waarom
elk transactiebericht verplicht een stuk "PSB‑achtige" content meedraagt.

Primaire bronnen (NL, normatief):
- `VISI 1.6/Documentation/VISI syst.1.6 Bijlage 3 Systematiek Deel 2 Berichten CC-BY-NC-SA.docx` — datamodel van berichten (EXPRESS)
- `VISI 1.6/Documentation/VISI syst.1.6 Bijlage 8 Richtlijn VISI communicatie met SOAP CC-BY-NC-SA.docx` — SOAP‑transport en scenario

## De sleutel: Type vs Template

VISI maakt een normatief onderscheid tussen definitie en instantie:

| In het raamwerk (definitie) | In een bericht (instantie)     |
|---|---|
| `MessageType`               | `MessageTemplate`              |
| `MessageInTransactionType`  | `MessageInTransactionTemplate` |
| `TransactionType`           | `TransactionTemplate`          |
| `TransactionPhaseType`      | `TransactionPhaseTemplate`     |
| `ProjectType`               | `ProjectTypeInstance`          |
| `OrganisationType`          | `OrganisationTemplate`         |
| `PersonType`                | `PersonTemplate`               |
| `RoleType`                  | `RoleTemplate`                 |
| `GroupType`                 | `GroupTemplate`                |
| `ComplexElementType`        | `ComplexElementTemplate`       |
| `AppendixType`              | `AppendixTemplate`             |
| *(geen Type)*               | `PersonInRole`, `AppendixGroup`|

"Template" in VISI‑jargon betekent dus **"instantie die in een bericht zit"**,
niet "blauwdruk". Dat is een taalkundig ongelukkige keuze die de hele opbouw
aanvankelijk troebel maakte — maar als je dit onderscheid eenmaal ziet, valt
alles op zijn plek.

Bijlage 3 §1.5 (MessageInTransactionTemplate) legt expliciet het
bestaansrecht uit:

> *"Dit is de entiteit die het mogelijk maakt de feitelijke
> MessageInTransactionType mee te nemen in het bericht. Hierdoor is de
> positie in de workflow van een transactie altijd eenduidig te achterhalen."*

→ Templates bestaan opdat elk bericht op zichzelf traceerbaar is zonder
externe context. Zelf‑herleidbaarheid is ontwerpdoel, geen conventie.

## Verplichte inhoud van een MessageTemplate (EXPRESS)

Uit Bijlage 3 §1.6, regels 237‑248 (non‑OPTIONAL velden zijn verplicht):

```
ENTITY MessageTemplate;
  identification                 : STRING;                        -- verplicht
  dateSend                       : DATETIME;                      -- verplicht
  dateRead                       : OPTIONAL DATETIME;
  state                          : OPTIONAL STRING;
  dateLaMu                       : OPTIONAL DATETIME;
  userLaMu                       : OPTIONAL STRING;
  initiatingTransactionMessageID : OPTIONAL STRING;
  initiatorToExecutor            : BOOLEAN;                       -- verplicht
  messageInTransaction           : MessageInTransactionTemplate;  -- verplicht, ingebed
  transaction                    : TransactionTemplate;           -- verplicht, ingebed
  template                       : ComplexElementTemplate;        -- verplicht, ingebed
END_ENTITY;
```

De laatste drie zijn cruciaal: die zijn getypeerd als *embedded Template
entities*, niet als string‑idrefs. Dat betekent semantisch "een volledige
instantie van dat type hoort hier bij", niet "een verwijzing naar iets
elders". In de XML‑serialisatie worden die instanties gefloet naar het
root‑niveau van `visiXML_MessageSchema` en verwijst de ouder via
`<XRef idref="..."/>` — maar dat blijft een embedded object in het datamodel.

## De transitieve sluiting (modelregel, geen stijladvies)

Uitgaand van `MessageTemplate` en alleen de verplichte velden volgend:

```
MessageTemplate
├── messageInTransaction : MessageInTransactionTemplate      [verplicht]
├── transaction : TransactionTemplate                        [verplicht]
│   ├── number, name, description, startDate, endDate       (§1.13)
│   ├── initiator : PersonInRole                             [verplicht]
│   │   ├── contactPerson : PersonTemplate                   [verplicht]
│   │   ├── organisation : OrganisationTemplate              [verplicht]
│   │   │   └── contactPerson : PersonTemplate               [verplicht]
│   │   └── role : RoleTemplate                              [verplicht]
│   ├── executor : PersonInRole (zelfde subboom)             [verplicht]
│   └── project : ProjectTypeInstance                        [verplicht]
│       └── template : ComplexElementTemplate                [verplicht]
└── template : ComplexElementTemplate                        [verplicht]
```

Dus elk transactiebericht bevat verplicht: het MessageType, de MITT, de
Transaction, het Project, twee Organisations (afzender + ontvanger), minstens
twee Persons (contactpersonen van beide organisaties, die kunnen samenvallen
met initiator/executor), twee Roles en twee PersonInRoles. Niet als
"PSB‑kopie voor de zekerheid" maar omdat de EXPRESS‑schema van Bijlage 3
deze velden als non‑OPTIONAL Template‑velden typeert. Implementaties hebben
geen vrijheid om dit weg te laten.

Nuance: je neemt alleen de **transitieve sluiting** vanaf deze ene message
mee, niet de hele PSB. Een PSB kan bijvoorbeeld 20 PersonInRoles bevatten;
een concreet bericht heeft er doorgaans 2 nodig (initiator + executor).
Andere rollen, personen of organisaties die niet in de keten van dit bericht
zitten, hoeven niet mee. Dat verklaart waarom verschillende berichten binnen
hetzelfde testproject verschillende subsets bevatten (bv. `zevende_bericht.xml`
in Top Koks heeft minder PersonInRoles dan `eerste_bericht.xml`).

## Normatieve tekstuele onderbouwing (Bijlage 8)

Drie plekken waar Bijlage 8 het expliciet maakt, in oplopende hardheid:

**§3 Scenario berichtuitwisseling, randvoorwaarde 3** (opening van hoofdstuk 3,
in mijn extract regel 106):

> *"Alle informatie over de aanwezige configuratie, URL adressen van
> personen in een bepaalde rol e.d. zijn gevat in VISI berichten volgens
> het raamwerk voor het uit te voeren project."*

Dit is *normatief* en de ontwerpbasis van het hele protocol. "Gevat in
VISI berichten" — niet in een cache, niet out‑of‑band, maar in het bericht.

**§3.2.1 Gevolgen** (regel 202):

> *"De gevolgen van deze aanpak is dat het informatie systeem (IS) in staat
> is bij elk bericht binnen een transactie te achterhalen welke URL behoort
> tot de afzender en welke URL behoort tot de ontvanger."*

"Bij elk bericht" — dus ook vervolgberichten, niet alleen het eerste.

**§3.4 opening** (regel 207):

> *"Deze beschrijving is geschikt voor alle type berichten binnen
> transacties, zowel een eerste bericht binnen een transactie als reacties
> op ontvangen berichten binnen een transactie."*

Geen apart scenario voor "reactie" — dezelfde regels gelden.

## Eerste bericht vs vervolgbericht

**Inhoud is identiek** (zelfde Template‑transitieve sluiting). **Bron
verschilt.** Bijlage 8 §3.4 stap 1 (regel 214):

> *"Het VISI bericht wordt opgemaakt door het versturende IS op basis van
> het ontvangen bericht (in geval van een nieuwe transactie wordt de
> informatie uit het projectspecifieke bericht gehaald)."*

| Situatie | Bron van ingebedde Templates |
|---|---|
| Eerste bericht nieuwe transactie | Lokaal gecachte PSB                 |
| Vervolgbericht lopende transactie | Het voorgaande, ontvangen bericht  |

Gevolg: zodra een conversatie loopt, draagt die zichzelf. De vervolgzender
raadpleegt niet zijn PSB‑cache maar neemt de relevante Templates
(project, beide PersonInRoles + subbomen, transaction) uit het binnengekomen
bericht over, en plakt er zijn nieuwe MessageTemplate + MITT + eventuele
nieuwe ComplexElements/Appendices aan. De transitieve sluiting blijft
daardoor automatisch behouden.

Dit heeft een belangrijke implicatie voor PSB‑updates tijdens een lopende
transactie. Bijlage 8 §4 (regel 336):

> *"De applicaties zullen bestaande transacties en nieuwe 'subtransacties'
> via het oude raamwerk laten lopen. Nieuwe transacties zullen via het
> nieuwe raamwerk opgestart worden."*

Een lopende transactie is dus functioneel bevroren op het raamwerk + PSB
beeld zoals in haar initiële bericht ingebed. Een PSB‑update midden in een
transactie beïnvloedt die transactie niet — de vervolgzender kopieert uit
het vorige bericht, niet uit zijn mogelijk vernieuwde cache.

## Serialisatie naar XML

De XSD‑root `visiXML_MessageSchema` is in elke gegenereerde raamwerk‑XSD een
`<xsd:choice maxOccurs="unbounded">` over **alle** entiteit‑elementen (zie
bv. `testproject/topkoks/10.xsd:6-83`). Dat betekent schema‑technisch: elk
entiteitstype mag in elke volgorde en elk aantal op root‑niveau voorkomen.
Dit is niet "laxheid" van de XSD maar de geserialiseerde vorm van het
EXPRESS‑model: embedded entities worden gefloet naar het root‑niveau, en
referenties vanuit de ouder gebruiken `<XRef idref="..."/>`.

Zo zie je in `testproject/topkoks/voorbeeld_bericht.xml`:

```xml
<msgWiltuDeKaartZien id="bericht001">
  ...
  <transaction>
    <t1_OpnameBestellingRef idref="transactie001"/>   <!-- verwijzing -->
  </transaction>
</msgWiltuDeKaartZien>
...
<t1_OpnameBestelling id="transactie001">              <!-- embedded object, geflot naar root -->
  ...
  <initiator><PersonInRoleRef idref="PiR002"/></initiator>
  <executor><PersonInRoleRef idref="PiR001"/></executor>
  <project><ProjectType1Ref idref="Project1"/></project>
</t1_OpnameBestelling>
<ProjectType1 id="Project1"> ... </ProjectType1>
<PersonInRole id="PiR001"> ... </PersonInRole>
<PersonInRole id="PiR002"> ... </PersonInRole>
<StandardOrganisationType id="consument"> ... </StandardOrganisationType>
<StandardOrganisationType id="restaurant"> ... </StandardOrganisationType>
<StandardPersonType id="KeesDeVries"> ... </StandardPersonType>
<StandardPersonType id="PietJansen"> ... </StandardPersonType>
<klant id="klantRol"> ... </klant>
<ober id="oberRol"> ... </ober>
```

Al deze root‑level entities zijn geen "los bijgevoegde PSB‑kopie" maar
de verplichte embedded objecten van `MessageTemplate` → `TransactionTemplate`
→ `PersonInRole` → `OrganisationTemplate`/`PersonTemplate`/`RoleTemplate`.
Zonder hen zou het EXPRESS‑datamodel stuk zijn.

## De relatie met het PSB

Een PSB (`project_specifiek_bericht_X.xml`) is zelf óók een `visiXML_MessageSchema`
document. Het verschilt in één opzicht van een transactiebericht: het bevat
alleen configuratie‑entities (ProjectTypeInstance, Organisations, Persons,
Roles, PersonInRoles) en **geen** MessageTemplate/TransactionTemplate/MITT.
Een transactiebericht bevat beide: de messaging‑kant + een
transitieve‑sluiting‑slice van de PSB.

Schema‑technisch bestaat er geen "PSB type" versus "message type". Allebei
zijn `visiXML_MessageSchema` instanties die verschillende subsets van
dezelfde choice invullen. De naam "projectspecifiek bericht" duidt op de
functie (bootstrap configuratie‑distributie), niet op een ander bestandstype.

De PSB wordt bij initialisatie éénmalig gedownload van een statische URI
(Bijlage 8 §3.3, regel 205). Daarna wordt hij in elk nieuw opgestart
transactie‑bericht deels gekopieerd (alleen de relevante deelgraaf). PSB‑updates
gaan via een formeel distributieproces met volgnummer en ingangsdatum
(Bijlage 8 §5, regel 340‑344), niet in‑band via transactieberichten.

## Waar komen de SOAP‑URLs precies vandaan, en waarom dubbel?

De `<SOAPServerURL>` en `<SOAPCentralServerURL>` uit de SOAP‑header zien er
hetzelfde uit als de `sOAPServerURL`/`sOAPCentralServerURL` velden in het
VISI‑bericht zelf (onder `ProjectType1.ceSOAP` respectievelijk
`StandardOrganisationType.ceOrganisatie`). Waarom beide?

Drie redenen uit Bijlage 8:

1. **SOAP‑server mag VISI niet parsen** (§3 randvoorwaarde 1, regel 104):
   *"De SOAP Servers en de SOAP Central Server(s) zijn niet in staat VISI
   berichten te parsen."* Hij kan de URLs dus niet zelf uit de body halen —
   het IS haalt ze eruit en overhandigt ze apart als header‑metadata.

2. **Body is de bron van waarheid, header is afgeleid** (§3.4 stap 2,
   regel 215): *"De URL adressen van het versturende en het ontvangende IS
   worden door het versturende IS uit het opgemaakte bericht gehaald."*
   Richting is eenduidig body → header.

3. **Zelf‑herleidbaarheid** (§3 randvoorwaarde 3, zie boven): alle
   configuratie‑info moet in het bericht zitten, dus ook de URLs. Dit laat
   de ontvanger toe om eventuele drift met zijn eigen PSB te detecteren,
   hoewel Bijlage 8 een dergelijke check niet expliciet eist.

De duplicatie is dus een gevolg van gelaagdheid (dom SOAP‑transport onder
semantisch‑rijke VISI‑body), geen redundantie die weggepoetst kan worden.

## Validatie aan de ontvangende kant

Bijlage 8 is hier opvallend dun. Wat er wél staat:

- **§3.4 stap 7** (regel 253‑254): *"Het IS van de ontvangende partij
  interpreteert het VISI bericht en indien akkoord verstuurt hij dit
  bericht + URL adres ..."*. Er is een "akkoord‑check", maar wát die
  precies inhoudt wordt niet genormeerd.
- **§3.4 stap 9 + foutvoorbeelden** (regel 295‑315): reactiebericht bevat
  een `<ERRORS>`‑blok met `<ERROR CODE="...">`‑elementen. De voorbeelden
  noemen expliciet *"bij validatie van de xsd"* met meldingen als
  *"Waarden van simpel element1 is niet volgens definitie"*. XSD‑validatie
  is dus normatief.
- **§3.4 stap 11** (regel 319): *"controleer of de informatie overeenkomt
  met het verstuurde bericht"* — round‑trip‑check bij de oorspronkelijke
  verzender, vermoedelijk op UniqueID + sender/receiver.

Wat er **niet** staat:
- Geen verplichting om de inline project/organisation/person data te
  vergelijken met een lokale PSB‑cache.
- Geen regel over wat te doen als de afzender een oudere PSB‑versie lijkt
  te gebruiken.
- Geen standaard foutcodes > 1 (§3.4 opmerking, regel 322): *"Tijdens het
  uitkomen van dit protocol zijn er geen standaard fout codes hoger dan 1
  gedefinieerd."*

Dus: normatief minimum = XSD‑valideren + integere round‑trip. Alles
daarboven (semantische checks, PSB‑drift‑detectie) is aan de implementeur.

## Korte samenvatting

- Elk transactiebericht bevat per EXPRESS‑definitie (Bijlage 3) een
  verplichte, transitieve sluiting van `MessageTemplate` → `TransactionTemplate`
  → `PersonInRole` → `OrganisationTemplate`/`PersonTemplate`/`RoleTemplate`
  → `ProjectTypeInstance` → etc.
- Dat is geen kopie van de hele PSB, maar de deelgraaf die nodig is om
  precies dít bericht in zijn workflow‑context eenduidig te kunnen plaatsen.
- De inname is normatief (Bijlage 8 §3 randvoorwaarde 3 + Bijlage 3
  datamodel). Niet optioneel.
- Eerste bericht put uit gecachte PSB; vervolgbericht put uit het voorgaande
  ontvangen bericht. Inhoud is identiek gestructureerd.
- Lopende transacties zijn bevroren op hun originele raamwerk + PSB‑beeld.
- SOAP‑header dupliceert de URLs uit de body omdat de SOAP‑server de body
  niet mag parsen; de body blijft de bron van waarheid.
- Ontvangst‑validatie is normatief beperkt tot XSD‑checks; semantische
  checks zijn impliciet en implementatie‑afhankelijk.
