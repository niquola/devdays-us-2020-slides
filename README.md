# Access Control

[Client] 
  -> [Auth Middleware] configured with AccessPolicy resource
    -> []

## Custom resources

* User
* Role
* AuthClient

```yaml
resourceType: User
name: niquola
roles: ['patient']
patient: {id: pt-1, resourceType: 'Patient'}

```

## Request Document

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

Different engines:

* JSON Schema and matcho
* SQL schema
* combination


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

### examples

JSON schema:
```yaml
resourceType: AccessPolicy
description: auth users
engine: json-schema
schema:
  type: object
  properties:
    uri: { pattern: '/Patiient/.*' } 
    request-method: { const: 'get' }
  required: ["user"]

```

Matcho:
```yaml
resourceType: AccessPolicy
engine: matcho
matcho:
  user: 
    role: admin
    data: {practitioner_id: present?}
  uri: '#/Encounter.*'
  request-method: {$enum: ['get', 'post']}
  params:
    practitioner: .user.data.practitioner_id
```

## Debug is important
