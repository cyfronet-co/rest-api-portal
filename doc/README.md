# Portal REST API
Following REST Endpoints are available:
  * `/api/sites/:site_key/grants` - returns information about all grants
    important for the site (drafts and grants in negociation are not returned)
  * `/api/sites/:site_key/allocations` - returns information about all
    allocations important for the site (drafts and allocations in negociation
    are not returned)
  * `/api/sites/:site_key/groups` - returns information about all
    groups important for the site.
  * `/api/sites/:site_key/users` - returns information about all users important for the site.
  * `/api/services/:service_key/users` - returns information about all users in service

## Authentication
Each site and service is secured by separate pair of keys which allow to generate signed JWT
token. Admin needs to upload the public key to the portal, private key
should be keep secured and it should be used to generate JWT tokens.

Use the following commands to generate new private/public key:

```
openssl ecparam -genkey -name secp521r1 -noout -out ecdsa-p521-private.pem
openssl ec -in ecdsa-p521-private.pem -pubout -out ecdsa-p521-public.pem
```


Next you can use any JWT library to generate JWT token.
The generated token should be passed as a `Authorization` bearer value.
 Bellow you can find example python program which generates token and
print `curl` invocations examples:

```bash
pip install pyjwt
```

```python
import jwt
import datetime

def generate_curl(site):
    with open(site + '.key', 'rb') as f:
        private_key = f.read()
    exp = (datetime.datetime.now() + datetime.timedelta(minutes=10)).strftime('%s')
    encoded = jwt.encode({'name': site, 'exp': exp}, private_key, algorithm='ES512')

    print('curl --header "Authorization:Bearer {}" https://grants.pre.plgrid.pl/api/sites/{}/grants'.format(encoded, site))
    print('curl --header "Authorization:Bearer {}" https://grants.pre.plgrid.pl/api/sites/{}/allocations'.format(encoded, site))


if __name__ == '__main__':
    generate_curl('CYFRONET-PROMETHEUS')
```

### Testing Site REST API in development mode
When `db:seed` is used example sites are generated. For each site script
generates public key (stored in the DB) and private key (stored in the `tmp`
directory). The key can be used to test site REST API by using `curl` commands.
We have a dedicated `rake` task which simplify this process and generate example
curl invocations:

```
./bin/rails jwt:token[site_key]

#e.g.
./bin/rails jwt:token[CYFRONET-PROMETHEUS]
```


## Grants REST API
Grants REST API returns list of all Grants important for the site. By important
we are understand grants in the following statuses: `accepted`, `blocked`,
`finished`, `settled` with allocations on the site.

### Endpoint
```
/api/sites/:site_key/grants
```

### Schema
```
[
  {
    name: String,
    author: String,
    title: String,
    description: Text,
    oecd: String,
    pilot: Boolean,
    start: Date,
    end: Date,
    lastChange: Time,
    team: String,
    status: Enum(accepted, blocked, finished, settled),
    relatedProjects: [
      {
        project_id: String,
        title: String,
        url: String,
        amount: Number,
        source: String,

      },
      ...
    ]
  },
  ...
[
```

### Statuses meanings
  * `accepted` - grant is accepted by the operator and has one or more
    allocations on the site. Grant should be active on the side in the
   `start`-`end` interval.
  * `blocked` - grant was blocked by the operator, all allocations should be
    blocked as well.
  * `finished` - grant is finished and it is in the settlement state
  * `settled` - grant settlement was send by the user and accepted by the
    operator.

## Allocations REST API
Allocations REST API returns list of all allocations important to the site. By
important we are understand site allocation from the grant important to the site
(see the previous section) in the following statuses: `accepted`, `blocked`.
`start`-`end` interval should be taken into account  to check if the
allocation is active on the site.

### Endpoint
```
/api/sites/:site_key/allocations
```

### Schema
```

[
  {
    name: String,
    grantName: String,
    resource: String,
    site: String,
    pilot: Boolean,
    start: Date,
    end: Date,
    lastChange: Time,
    status: Enum(accepted, blocked),
    parameterValues: [
      {
        key: String,
        value: String/Number
      },
      ...
    ]
  },
  ...
]
```
### Statuses meanings
  * `accepted` - allocation is accepted by the site admin and should be active
    in the `start`-`end` interval.
  * `blocked` - allocation grant was blocked by the operator or site admin


## Groups REST API

Groups REST API returns list of all groups that have at least grant with allocation on particular site.

The Groups API is versioned using a header parameter.
That means, requested version have to be passed in specific format as a header parameter
(e.g. `Accepted: v1.0` for the first API version).
If no version is specified or the format is wrong, the first (v1.0) version of API is returned.

### Endpoint
```
/api/sites/:site_key/groups
```

### Schema

#### v1.0:

```
    {
      "teamId": "plgguser1group",
      "name": "plgguser1group",
      "status": "accepted",
      "type": "private",
      "teamLeaders": [
        "plguser1",
        "plgroot"
      ],
      "teamMembers": [
        "plgroot",
        "plguser1"
      ]
    }
```

#### v2.0:
```
    {
      "group_id": String,
      "description": String,
      "type": String,
      "status": String,
      "members": [
        {
          "login": String,
          "cuid": String
          "role": String
        },
      ]
    },
```


## Users REST API

Users REST API returns list of all users that have at least one grant with allocation on particular site.

### Endpoint
```
/api/sites/:site_key/users
/api/sites/:site_key/user/:login
```

### Schema

```
    {
      "id": Integer,
      "email": String,
      "firstName": String,
      "lastName": String,
      "login": String,
      "status": String,
      "affiliationList": [
        {
          "type": String,
          "units": [
            String,
            String
          ]
        }
      ]
    }
```

## Service users REST API

Service users REST API returns list of all users in service.

### Endpoint
```
/api/services/:service_key/users
```

### Schema

```
    {
      "login": String,
      "state": String,
      "motivation": String
    }
```

