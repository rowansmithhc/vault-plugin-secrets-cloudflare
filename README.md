# Vault Secrets Plugin - Cloudflare

[Vault][vault] secrets plugins to simplify creation, management, and
revocation of [Cloudflare][cloudflare] API tokens.

This is a fork of https://github.com/bloominlabs/vault-plugin-secrets-cloudflare, created because
I wanted to see a version with the following changes:
1. Built with go 1.21
2. Built with cloudflare-go v0.81.0
3. Built with vault/api v1.10.0
4. Built with vault/sdk v0.10.2
5. Improved the README to support usability.

## Usage

### Configure Endpoint

1. Download and enable plugin locally (TODO)

2. Configure the plugin with a token capable of generating other tokens:

```
vault write /cloudflare/config/token token=<token>
```

### Configure Policies

Note when creating policies the Cloudflare API docs are of help:
* Token Creation information: https://developers.cloudflare.com/api/operations/user-api-tokens-create-token
  * Note that only the contents of the `policies` block is required, not the `name` or `condition` block etc.
* The `id` values in the `permission_groups` correspond to the relevant permission groups on Cloudflare. The doc at https://developers.cloudflare.com/api/operations/permission-groups-list-permission-groups 
can be used as a reference to retrieve a list of values to use.
* The resources block references the domains on which the permissions are applied.
* A list of zones and their details can be retrieved
  via the Cloudflare API, see https://developers.cloudflare.com/api/operations/zones-get for an example.

An example policy which allows for reading and editing of DNS records on a zone with the id
`069d3066870c958bad1cd2a767b78g86` is included below. 

1. Create a role and supply an appropriate policy:

```
vault write /cloudflare/roles/<role-name> policy_document=-<<EOF
[
  {
        "effect": "allow",
        "permission_groups": [
          {
            "id": "82e64a83756745bbbb1c9c2701bf816b",
            "name": "DNS Read"
          },
          {
            "id": "4755a26eedb94da69e1066d98aa820be",
            "name": "DNS Write"
          }
        ],
        "resources": {
          "com.cloudflare.api.account.zone.069d3066870c958bad1cd2a767b78g86": "*"
        }
  }
]
EOF
```

### Generate a Cloudflare token using the role:

```
vault read /cloudflare/creds/<role-name>
```

### Rotating the Root Token

The plugin supports rotating the configured admin token to seamlessly improve
security.

To rotate the token, perform a 'write' operation on the
`config/rotate-root` endpoint

```bash
> export VAULT_ADDR="http://localhost:8200"
> vault write -f config/rotate-root
Key      Value
---      -----
name     vault-admin-{timestamp in nano seconds}
```

### Generate a new Token

To generate a new token:

[Create a new cloudflare policy](#configure-policies) and perform a 'read' operation on the `creds/<role-name>` endpoint.

```bash
# To read data using the api
$ vault read cloudflare/role/dns-edit
Key                Value
---                -----
lease_id           cloudflare/creds/test/956Fo9MQgleoqosK5wuMVwPC
lease_duration     768h
lease_renewable    true
id                 9c40db059267e91c7f3f22220c1536ed
token              <token>
```

## Development

The provided [Earthfile] ([think makefile, but using
docker](https://earthly.dev)) is used to build, test, and publish the plugin.
See the build targets for more information. Common targets include

```bash
# build a local version of the plugin
$ earthly +build

# execute integration tests
#
# use https://developers.cloudflare.com/api/tokens/create to create a token
# with 'User:API Tokens:Edit' permissions
$ TEST_CLOUDFLARE_TOKEN=<YOUR_CLOUDFLARE_TOKEN> earthly --secret TEST_CLOUDFLARE_TOKEN +test

# start vault and enable the plugin locally
earthly +dev
```

[vault]: https://www.vaultproject.io/
[cloudflare]: https://www.cloudflare.com/
[earthfile]: ./Earthfile
