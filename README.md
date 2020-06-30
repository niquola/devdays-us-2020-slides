# Access Control in Aidbox

```
[Client] 
  -> [Authz Middleware] configured with AccessPolicy resource
    -> [response or deny]
```

## Custom resources

* User (SCIM like)
* Role
* AuthClient
* Request

```yaml
resourceType: User
name: niquola
roles: ['patient']
patient: {id: pt-1, resourceType: 'Patient'}

```

## Request as a Document/Resource

```yaml
request-method: get

# Request URI (no query string)
uri: /fhir/Patient

params: 
  type: Patient
  name: john
  _include: practitioner

# Object containing current User resource (if any)
user:
  resourceType: User
  patient: {id: pt-2, resourceType: Patient}
  roles: ['practitiner]
  email: foo@foo.com
  phone: 123-123-123
  ...
cleint:
  resourceType: AuthClient
  id: cl-1
  ...
jwt: {sub: xxxxxxx, jti: xxxxxxx, iss: aidbox,
  iat: 15409xxxxx, exp: 15409xxxxx}

body:
 ...

```


## AccessPolices

AccessPolicy is a function:
```
policy(request) => {allow/deny, explain?}

evan(policy,request) => {allow/deny, explain?}

# chain
for each policy:
  Eval(policy, request) => {allow/deny, explain?}
```

Represented as custom resource

```yaml
resourceType: AccessPolicy
description: policy description text
engine: allow | sql | json-schema | complex

roleName: practitiner
link:
- { resourceType: Operation, id: op-id }
- { resourceType: User, id: user-1 }
- { resourceType: Client, id: client-1 }
...
```

Evaluation order:

* Operation Policies
* User/Client Policies
 * if user has roles: Role Policies
* Global Policies

Untill first allows!

Different engines:

* JSON Schema and matcho
* SQL schema
* combination



### examples

JSON schema:
```yaml
type: object
properties:
  uri: { pattern: '/Patient/.*' } 
  request-method: { const: 'get' }
required: ["user"]
```

Matcho:
```yaml
user: 
  role: admin
  data: {practitioner_id: present?}
uri: '#/Encounter.*'
request-method: {$enum: ['get', 'post']}
params:
  practitioner: .user.data.practitioner_id
```

SQL:
```sql
SELECT
  {{user}} IS NOT NULL
  AND {{user.data.practitioner_id}} IS NOT NULL
  AND {{uri}} LIKE '/fhir/Patient/%'
  AND resource#>>'{generalPractitioner,id}' = {{user.data.practitioner_id}}
FROM patient WHERE id = {{params.resource/id}};
```

Clojure policy:
```
(fn [ctx req]
  ...)
```

## Debug & Introspection are important

Not only predicate mode for policy,
but also an **explain mode**

+ explain in audit mode (per request)

```yaml
POST /auth/test-policy

request:
  uri: '/Patient'
  request-method: get
  headers:
    authorization: Bearer <your-jwt>
  user:
    data: {role: 'admin'}
  client:
    id: basic
    grant_types: ['basic']
policy:
  engine: sql
  sql:
    query: 'SELECT {{user.role}} FROM {{!params.resource/type}}'

-- response

operation:
  request:
  - get
  - {name: resource/type}
  action: proto.operations/search
  module: proto
  id: Search
-- original policy
policy:
  engine: sql
  sql: {query: 'SELECT {{user.role}} FROM {{!params.resource/type}}'}
-- result of policy evaluation
result:
  eval-result: false
  query: ['SELECT ? FROM "patient"', admin]
```

## Few tricks

* enforce specific params with policy


```yaml
user: 
  role: admin
  data: {practitioner_id: present?}
uri: '#/Encounter.*'
request-method: {$enum: ['get', 'post']}
params:
  practitioner: .user.data.practitioner_id <--HERE
```

Probably we need add _elements paramss for most of REST API! 
read and even create/update

```
GET /Patient/id?_elements

PATCH /Patient/id?_elements
```

## _elements extensions for includes

* nested elements: _elements=name.given
* (rev)includes elements: _elements=Patient.name

TODO: fhirpath elements (for masking)

```
_elements=identifier.where(system=ssn).value.mask()
```

TODO: custom transformation of response?

* HS Jute
* FHIR Mapping

## Compartments

A lot of clients are using our dynamic compartments to simplify Access Policies

```
resourceType: AccessPolicy
engine: matcho
matcho:
  uri: '#/patient/.*/'
  request-method: {$enum: ['get']}
  route-params:
    patient-id: .user.patient.id

```

There are request to **write** into compartments
i.e.:

```
POST /patient/<pid>/Observation

resourceType: Observation
...

```

## Design points

1. REST idempotence:  same URI => same resource!

* dont change response content implicitly
* dont inject implicit logic based on user

2. Poka-yoke: Ability to design API to simplify access control

* compartments one example
* implicit params & elements injection etc

3. Performance and cachability

* policy based on request
* policy based on data on server
* policy based on world state

TTL? safe-to-cache?

## API Construction as compartments on steroid

Define Custom Operations with set of rules
for implicit params, allowed params, transformations


```
resourceType: Operation
action: fhir-search
resourceType: Observation
mount: '/compartment/:patient-id/Observation'
implicit-params:
  subject: .params.patient-id
  _security: http://...|non-sensitive 
masking:
- .identifier
- .name.family
allow-params:
  code: ... 
  ... 
allow-includes:
  patient:
    elements: {name: {}, birthDate: {}}

```

```
resourceType: Operation
action: fhir-create
resourceType: Observation
mount: '/compartment/:patient-id/Observation'
implicit-elements:
  subject: {id: .params.patient-id, resourceType: Patient }
  _security: [{system: http://... code: non-sensitive}
  meta: {source: '_patient'}
```

```
resourceType: Operation
action: fhir-patch
resourceType: Observation
mount: '/nurse/:pract-id/Encounter'
allow-elements:
  status: {...}
```
