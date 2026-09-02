---
name: carefluence-uscdi-patient-record-retrieval
description: >-
  Pull a patient's USCDI record set out of the Carefluence FHIR R4 server —
  find the patient, then walk the clinical resources in their compartment,
  paging correctly and honouring the read-only scope model.
api: Carefluence Open API R4
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the live CapabilityStatement at
  https://classic.carefluence.com/r4/metadata (saved at
  fhir/carefluence-openapi-r4-capabilitystatement.json) and the verbatim request
  set in the published Postman collection at https://api.carefluence.com/
  (saved at postman/carefluence-openapi-r4-collection.json). Every path and
  search parameter below is declared by that CapabilityStatement or published in
  that collection; none is invented.
operations:
  - 'GET {base}/metadata'
  - 'GET {base}/Patient?name={name}'
  - 'GET {base}/Patient?identifier={system}|{value}'
  - 'GET {base}/Patient?name={name}&birthdate={date}'
  - 'GET {base}/Patient/{id}'
  - 'POST {base}/Patient/_search'
  - 'GET {base}/Condition?patient={id}'
  - 'GET {base}/AllergyIntolerance?patient={id}'
  - 'GET {base}/MedicationRequest?patient={id}'
  - 'GET {base}/Immunization?patient={id}'
  - 'GET {base}/Procedure?patient={id}'
  - 'GET {base}/Encounter?patient={id}'
  - 'GET {base}/CarePlan?patient={id}'
  - 'GET {base}/CareTeam?patient={id}'
  - 'GET {base}/Goal?patient={id}'
  - 'GET {base}/Device?patient={id}'
  - 'GET {base}/DiagnosticReport?patient={id}&category={code}&date={date}'
  - 'GET {base}/DocumentReference?patient={id}&type={loinc}'
  - 'GET {base}/Observation?patient={id}&category={category}&code={loinc}'
  - 'GET {base}/Patient?_id={id}&_revinclude=Provenance:target'
---

# Retrieve a USCDI patient record from Carefluence

`{base}` is `https://classic.carefluence.com/r4/` — the value the server's own
CapabilityStatement declares as `implementation.url`. `https://fhir.carefluence.com/r4/`
serves the identical contract. Get a bearer token first
(`carefluence-smart-app-authorization`).

## 1. Confirm the contract before you call it

`GET {base}/metadata` is anonymous and returns the CapabilityStatement. Read
`rest[0].resource[]` for the exact search parameters each resource accepts — they
are not uniform, and several resources accept far fewer than FHIR allows.

## 2. Find the patient

Published, working search forms:

- `GET {base}/Patient?name=Chalmers`
- `GET {base}/Patient?identifier=12345`
- `GET {base}/Patient?identifier=urn:oid:1.2.36.146.595.217.0.1|12345`
- `GET {base}/Patient?name=Chalmers&birthdate=1974-12-25`
- `GET {base}/Patient?name=Chalmers&gender=male`
- `GET {base}/Patient/{id}` and `GET {base}/Patient?_id=4`
- `POST {base}/Patient/_search` when the query is too long or too sensitive for a URL

Patient's declared search parameters are exactly `_id, identifier, name, gender,
birthdate`. Anything else will not filter.

## 3. Walk the compartment

Twelve resource types accept `?patient={id}`: `Condition`, `AllergyIntolerance`,
`MedicationRequest`, `Immunization`, `Procedure`, `Encounter`, `CarePlan`,
`CareTeam`, `Goal`, `Device`, `DiagnosticReport`, `Observation` (plus
`AuditEvent` and `MedicationStatement`). Request one scope per resource type you
intend to read; `patient/*.read` covers the compartment in one grant.

## 4. Observations need a code or a category

`Observation` is the resource that carries most of USCDI, split by profile. Use
`code` (LOINC) or `category` to select the slice you want, and combine with
`date` ranges using FHIR prefixes:

- Smoking status: `?patient={id}&code=72166-2`
- Vitals: `?patient={id}&category=vital-signs&code={loinc}` — body height `8302-2`,
  body weight `29463-7`, body temperature `8310-5`, heart rate `8867-4`,
  respiratory rate `9279-1`, blood pressure `85354-9`, pulse oximetry `2708-6`
- Pediatric growth: BMI-for-age `59576-9`, weight-for-height `77606-2`,
  head circumference percentile `8289-1`
- Labs: `?patient={id}&category=laboratory&code={loinc}` (the published example
  uses `38483-4`)
- Date windows: `&date=gt2020-05-04&date=lt2021-05-06`

`DocumentReference` uses LOINC too: History & Physical `34117-2`, consultation
note `11488-4`, discharge summary `18842-5`, progress note `11506-3`.

## 5. Page the Bundle, do not assume you got everything

Responses are FHIR `Bundle`s. The server pages at `_count=20` by default and
expresses the next page through `Bundle.link` URLs carrying
`searchId`, `page`, `_count`, `startIndex` and `_total`. Follow the link the
server gives you; do not construct the offsets yourself.

## 6. Get provenance when you need to say where data came from

`GET {base}/Patient?_id=4&_revinclude=Provenance:target` returns the Patient plus
the Provenance resources that point at it — that is the direction the linkage
runs, and it is the published pattern.

## Constraints to design around

- **Read only, in practice.** The CapabilityStatement declares `create`, `update`
  and `patch` on 23 of 24 resource types, but the authorization server advertises
  no `.write` scope at all. Do not build a write path against this API without
  confirming with Carefluence that a write grant exists.
- **No delete, no history.** `conditionalDelete` is `not-supported` everywhere and
  no `vread`/`_history` interaction is declared, so there is no undo and no way to
  read a prior version. See the `reversibility` block in
  `conventions/carefluence-conventions.yml`.
- **No published rate limits.** Nothing tells you the ceiling or how you will be
  told you hit it. Throttle yourself conservatively.
- **`Provenance` and `MedicationRequest` advertise `_id, name, gender, birthdate`**
  as their search parameters — the Patient parameter set, copied onto resources it
  does not fit. Search those two by `patient` and by id and verify the results
  rather than trusting the declaration.
- **Two requests in the published collection point at `localhost`** (`:40927`,
  `:62749`). Substitute `{base}`.
