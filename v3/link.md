# Links

A `link` is the core connectivity mechanism between two endpoints.  An endpoint with one or more links is referred to as a `mesh`.

## Terminology

* **Link CSID** - The highest matching `CSID` between two endpoints
* **Link Keys** - The one or more `CSKs` of the other endpoint, at a minimum must include the `CSID` one
* **Link Paths** - All known or potential path information for connecting a link
* **Link Handshake** - A handshake that contains one `CSK` and the intermediate hashes for any others to validate the hashname and encrypt a response

## Link State

Links can be in three states:

* **unresolved** - at least the hashname is known, but either the Link Keys or Link Paths are incomplete
* **down** - keys have been validated and at least one path is available (possibly through router), but the link is not connected
* **up** - link has sent and received a handshake and is active

<a name="json" />
## JSON

Many apps use JSON as an easy storage/exchange format for defining and sharing link data.  The only required standard for this is that each link is an object with two fields, a `keys` object and a `paths` array.  Apps may extend and customize the JSON as needed but should attempt to preserve those two fields to simplify what other tools and libraries can automatically detect and generate.

It's common to have a `hashname` field as well for convenience or as the verified value if only one `CSK` is stored.

```js
  {
    "keys": {
      "1a": "aiw4cwmhicwp4lfbsecbdwyr6ymx6xqsli"
    },
    "paths": [
      {
        "ip": "192.168.0.55",
        "port": 61407,
        "type": "udp4"
      },
      {
        "hn": "e5mwmtsvueumlqgo32j67kbyjjtk46demhj7b6iibnnq36fsylka",
        "type": "peer"
      }
    ],
    "hashname": "frnfke2szyna2vwkge6eubxtnkj46rtctqk7g7ewbvfiesycbjdq"
  }
```

The `keys` object is always a dictionary of at least the single `CSID` for the link, with all string values being a [base32](hashname.md) encoding of the binary `CSK` for that given `CSID`.

The `paths` array is always the list of current or recent [path values](channels/path.md) and should contain only external paths when shared or a mix of both internal and external when used locally.

<a name="jwk" />
## JSON Web Key (JWK)

The Link Keys can also be represented in a standard [JWK](https://tools.ietf.org/html/draft-ietf-jose-json-web-key-41) using a `kty` of `hashname`:

```json
{
    "kty": "hashname",
    "kid": "27ywx5e5ylzxfzxrhptowvwntqrd3jhksyxrfkzi6jfn64d3lwxa",
    "use": "link",
    "cs1a": "an7lbl5e6vk4ql6nblznjicn5rmf3lmzlm",
    "cs3a": "eg3fxjnjkz763cjfnhyabeftyf75m2s4gll3gvmuacegax5h6nia"
}
```

The `kid` must always be the matching/correct hashname for the included keys.  The `use` value must always be `link` as it can only be used to create links.

The JWK may also contain a `"paths":[...]` array if required, often the JWK is only used as [authority validation](uri.md#discovery) and does not require bundling of the current link connectivity information.

## Resolution

Links can be resolved from any string:

1. [JSON](#json)
2. [Direct URI](uri.md) (no fragment)
3. [Peer URI](uri.md#peer) (router assisted, with fragment)
3. hashname - [peer request](channels/peer.md) to default router(s)

Once resolved, all paths should be preserved for future use.  If resolved via a router, also generate and preserve a `peer` path referencing that router.

<a name="handshake" />
## Handshake

The handshake packet is of `"type":"link"` and contains an optional `"csid":"1a"` for use when not sent as a message (such as in a [peer](channels/peer.md)).  The `BODY` of the handshake is another encoded packet that contains the sender's hashname details.

The attached packet must include the correct `CSK` of the sender as the `BODY` and the JSON contains the intermediate hash values of any other `CSIDs` used to generate the hashname.

Example:

```json
{
  "type":"link",
  "at":123456789,
  "csid":"2a"
}
BODY:
  {
    "3a": "eg3fxjnjkz763cjfnhyabeftyf75m2s4gll3gvmuacegax5h6nia",
    "1a": "ckczcg2fq5hhaksfqgnm44xzheku6t7c4zksbd3dr4wffdvvem6q"
  }
  BODY: [2a's CSK binary bytes]
```

<a name="jwt" />
## Identity (JWT)

The endpoints connected over a link are always uniquely identified by their hashnames, which serves as a stable globally unique and verifiable address, but is not intended to be used as a higher level identity for an end-user or other entity beyond the single instance/device.  Once a hashname is generated in a new context, it should be registered and associated with other portable identities by the application.

[OpenID Connect](http://openid.net/connect/) or any service that can generate a [JSON Web Token](http://tools.ietf.org/html/draft-ietf-oauth-json-web-token) can be used as the primary user/entity identification process, enabling a strongly encrypted communication medium to be easily coupled with standard identity management tools.

Just as a JWT is sent as a Bearer token over HTTP, it can be automatically included as part of the [handshake process](e3x/handshake.md) between endpoints.  This enables applications to require additional context before deciding to establish a link or apply restrictions on to what can be performed over the link once connected.

### Audience

When an [ID Token](http://openid.net/specs/openid-connect-basic-1_0.html#IDToken) is generated specifically for one or more known hashnames, the hashname must be included in the `aud` as one of the items in the array value.

### Scope

When a client is requesting to establish a new link to an identity, it must include the scope value `link` during authorization.

### Claims

An identity may advertise its connectivity by including a `link` member in the [Standard Claims](http://openid.net/specs/openid-connect-basic-1_0.html#StandardClaims).  The value must be a valid [URI](uri.md) that can be resolved to establish a link, and any resulting linked hashname must be included in the token's `aud` audience values.

