# did:web AnonCreds Method

## Goal

 Define a common way to access and reference AnonCreds objects that belong to a did:web controller.
 
 ## DID URLs
 
  In order to be compliant with existing DID Core and `did:web` Method Specifications, a standard mechanism for DID URLs referencing AnonCreds objects is used, based on specific services that must be defined in their issuer's DID Document. 
  
  A generic approach is to define a base AnonCreds service that serves for multiple objects created by a certain party. As a result, DID URL for a given object will have the following structure:
 
 ```
 <did>?service=<anoncreds-service-id>&relativeRef=/<object-path>/<object-id>
 ```

- `<did>` the did:web DID of the object-owning controller
- `<anoncreds-service-id>` the ID of the service (within the DIDDocument) containing the base endpoint to get the object
- `<object-path>` arbitrary, optional path (defined by the service) to get the object
- `<object-id>` unique identifier for the AnonCreds object on its scope
 
Example:

```
did:web:api.anoncreds.idlab.app:acme?service=schema&relativeRef=/5762v4VZxFMLB5n9X4Upr3gXaCKTa8PztDDCnroauSsR
```

An advantage about using this is that is "future proof" in case a different way of object resolution or identification is needed, while still allowing resolution of objects from a previous family version.

In addition, it allows to provide multiple base URLs, as the AnonCreds service in the DIDDocument can have more than a single `serviceEndpoint`.

A drawback for this approach is the verbosity of the URLs, which could become significant if exchanging several messages referencing these identifiers, or generating QR codes.

An alternative (yet compatible with current DID methods and URL conventions) that could be convenient for simple use cases is to use a specific service for each AnonCreds object, resulting a shorter DID URL:

 ```
 <did>#<object-id>
 ```

This case will imply that, upon the creation of a new AnonCreds object, the DID Document must be updated. This means that any AnonCreds object resolver willing to resolve an object from an issuer must ideally resolve the DID Document first in order to make sure it takes into account the latest version.

## Object creation

AnonCreds objects are created and stored in any suitable form for the underlying system. This specification does not enforce any particular API to do so.

However, due to the wide range of servers that may host `did:web` objects, some special measures are taken in the method in order to aid object resolvers to verify data integrity and/or authenticity.

### Hash as Object ID

Albeit there is not any restriction on the paths chosen to host AnonCreds resources, due to the objects are meant to be immutable, the <object-id> part of its URL is defined as a string calculated as the [Cryptographic Hyperlink](https://datatracker.ietf.org/doc/html/draft-sporny-hashlink) of object contents.

Due that AnonCreds objects are represented using JSON notation, before any calculation their contents must be canonized following [IETF RFC 8785](https://datatracker.ietf.org/doc/rfc8785/).

Example:
```json
{
  "issuerId": "did:web:api.anoncreds.idlab.app:acme",
  "name": "idlab_demo",
  "version": "0.0.1",
  "attrNames": ["id", "name"]
}
```

Once canonicalized, this object becomes:
    
```
{"attrNames":["id","name"],"issuerId":"did:web:api.anoncreds.idlab.app:acme","name":"idlab_demo","version":"0.0.1"}
```

Applying the SHA2-256, the following output is returned (in hex):
    
```
3cfdec646be07e4834a64eae2f9038e079585ce7536bf73e6e71a0899c673ba0
```

Finally, converting to base58 we get the object id:
    
```
5762v4VZxFMLB5n9X4Upr3gXaCKTa8PztDDCnroauSsR
```

And supposing the issuer DID includes an `anoncreds` service in their DID Document and uses `/v1/schema` as object path, the full object DID URL would be:
    
```
did:web:domain?service=anoncreds&relativeRef=/v1/schema/5762v4VZxFMLB5n9X4Upr3gXaCKTa8PztDDCnroauSsR
```

### Verifications before creating an object

Some verifications are needed before creating an object:

- Schema name and version should be unique for a particular scope?
- Other?


## Object resolution

Following the rules from [did:web method](https://w3c-ccg.github.io/did-method-web/) combined with concepts from [DID Core](https://www.w3.org/TR/did-core/), in order to dereference DID URLs that correspond to AnonCreds objects, some simple steps moust be followed:

1. Resolve object controller DID as specified in did:web
2. Find the referenced service in the DID Document and retrieve its `service-endpoint`
3. Perform an HTTPS GET request to `<service-endpoint>/<object-path>/<object-id>`
4. Parse the response as a AnonCreds Object Resolution JSON object

AnonCreds Object Resolution has the following structure, in a similar fashion that [DID Resolution](https://w3c-ccg.github.io/did-resolution/):

```json
{
 "resource" : {...},
 "resourceMetadata": {...}
}
```

If the object exists, the server shall respond with a status 200 and an object containing the structure specified above. If not found, it shall respond with a 404 status.

## Object types and their metadata
 
For the first AnonCreds object family (1.0), the following objects are identified:
- Schema
- Credential Definition
- Revocation Registry Definition
- Revocation Status List

This section describes the specifities of each type of object, especially in terms of their metadata contents.
    
### Schema

No specific object metadata is defined for schema.

### Credential Definition
 
No specific object metadata is defined for credential definition.

### Revocation Registry Definition
 
Each Revocation Registry must include in its metadata an endpoint that will be used to retrieve its different revocation status list entries:

```json
{
   "resourceMetadata": {
        "revocationStatusListEndpoint": "https://mydomain/revStatus/5762v4VZxFMLB5n9X4Upr3gXaCKTa8PztDDCnroauSsR"
    } 
}
```  
    
### Revocation Status List
    
Revocation Status lists are a special case, because they are not typically referenced by their ID but retrieved as the result of a query to the VDR for a given timestamp.
    
Therefore, issuers supporting revocable credentials must include endpoints for revocation status lists querying. The base URL for such endpoint must be attached as part of the `objectMetadata` for the related revocation registry definition.
    
Then, the active Revocation Status List for a given timestamp can be retrieved by performing an HTTPS GET to `<revocation-status-list-endpoint>/<timestamp>`.

In the response, the server shall attach the following fields to `objectMetadata`:
    
- **previousVersionId**: string representing Unix timestamp for the immediate previous revocation status list. "" if it's the first one
- **nextVersionId**: string representing Unix timestamp for the immediate next revocation status list. "" if it's the most recent one
    
```json
{
   "resourceMetadata": {
        "previousVersionId": "1680033457",
        "nextVersionId": "",
    } 
}
```  