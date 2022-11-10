# Portal REST API
Following REST Endpoints are available:
  * `/api/sites/:site_key/grants` - returns information about all grants
    important for the site (drafts and grants in negociation are not returned)
  * `/api/sites/:site_key/allocations` - returns information about all
    allocations important for the site (drafts and allocations in negociation
    are not returned)

## Authentication
Each site is secured by separate pair of keys which allow to generate signed JWT
token. Site admin needs to upload the public key to the portal, private key
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
def generate_curl(hostname, site):
    """
    Generate curls for API
    :param hostname: hostname of Grants system app
    :param site: site key example - CYFRONET-ZEUS
    :return: 
    """
    with open(site + '.key', 'rb') as f:
        private_key = f.read()
    exp = (datetime.datetime.now() + datetime.timedelta(minutes=10)).strftime('%s')
    encoded = jwt.encode({'name': site, 'exp': exp}, private_key, algorithm='ES512')
    print('curl --header "Authorization:Bearer {}" https://{}/api/sites/{}/grants'.format(encoded, hostname, site))
    print('curl --header "Authorization:Bearer {}" https://{}/api/sites/{}/allocations'.format(encoded, hostname, site))
if __name__ == '__main__':
    #generate_curl(hostname=, site=)
```

## Grants REST API
Grants REST API returns list of all Grants important for the site. By important
we are understand grants in the following states: `active`, `finished`, `closed`,
`in_renegotiation`, `blocked` with allocations on the site.

### Endpoint
```
/api/sites/:site_key/grants
```

### Schema
```
[
  {
    name: String,
    title: String,
    description: Text,
    start: Date,
    end: Date,
    lastChange: Time,
    team: String,
    state: Enum(active, finished, closed, in_renegotiation, blocked),
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
(see the previous section) in the following states: `accepted`, `blocked`,
`user_renegotiation`, `admin_renegotiation`, `ended`.

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
    start: Date,
    end: Date,
    lastChange: Time,
    state: Enum(accepted, blocked, user_renegotiation, admin_renegotiation, ended),
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
