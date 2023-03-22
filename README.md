# JABface-API-Doc
JABface API Documentation

  - [SwaggerUI](https://editor.swagger.io/?url=https://raw.githubusercontent.com/monkeyfirst-ch/JABface-API-Doc/main/jabface-api-doc.yaml)

## Security architecture

### Content signature

The content transferred to the REST API is signed with the private key of the certificate issued by "Monkeyfirst Regular CA 01".

Given the JSON payload to be sent (data used to create the JABface ticket or revoke), the process is as follows:

1. Primary system create a canonicalized text representation by removing all spaces, tabs, carriage returns and newlines from the payload. The regex `/[\n\r\t ]/gm` can be used.
2. Primary system encodes gets a UTF8 byte representation of that canonicalized text.
3. Primary system signs that byte representation using the algorithm "RSASSA-PKCS1-v1_5" from [RFC](https://datatracker.ietf.org/doc/html/rfc3447). Most implementations name the algorithm `SHA256withRSA`.
4. Primary system encodes the signature as base64 string.
5. Primary system places the base64 encoded signature in the request header `X-Signature`.

#### Java signature sample

```java
// load the key
PrivateKey privateKey = this.getCertificate();
// canonicalize
String normalizedJson = payload.replaceAll("[\\n\\r\\t ]", "");
byte[] bytes = normalizedJson.getBytes(StandardCharsets.UTF_8);
// sign
Signature signature = Signature.getInstance("SHA256withRSA");
signature.initSign(privateKey);
signature.update(bytes);
String signatureString = Base64.getEncoder().encodeToString(signature.sign());
```

#### .NET signature sample

```c#
// create RSA from certificate
X509Certificate2 cert = GetCertificate();
RSA rsaSignature = cert.GetRSAPrivateKey();

// normalize json
Regex normalizedJsonReplaceRegex = new Regex("[\\n\\r\\t ]");
string normalizedJson = normalizedJsonReplaceRegex.Replace(payload, string.Empty);

// sign
byte[] signatureBytes = rsaSignature.SignData(Encoding.UTF8.GetBytes(normalizedJson), HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);

// convert signature to Base64 string
string signatureString = Convert.ToBase64String(signatureBytes);
```

#### Node.js TypeScript signature sample

```typescript
// load the key
const pemEncodedKey = fs.readFileSync(privateKeyFile)
const privateKeyObject = crypto.createPrivateKey(pemEncodedKey)
// canonicalize
const regex = /[\n\r\t ]/gm
const canonicalPayload = payload.replace(regex, '')
const bytes = Buffer.from(canonicalMessage, 'utf8')
// sign
const sign = crypto.createSign('RSA-SHA256')
sign.update(bytes)
const signature = sign.sign(privateKeyObject)
const base64encodedSignature = signature.toString('base64')
// set request header
headers['X-Signature'] = base64encodedSignature
```

### Structure of the Payload

The payload is structured and encoded as a CBOR with a COSE digital signature. This is commonly known as a "CBOR Web Token" (CWT), and is defined in [RFC 8392](https://tools.ietf.org/html/rfc8392). The payload, as defined below, is transported in a JABface claim.

The integrity and authenticity of origin of payload data MUST be verifiable by the Verifier. To provide this mechanism, the issuer MUST sign the CWT using an asymmetric electronic signature scheme as defined in the COSE specification ([RFC 8152](https://tools.ietf.org/html/rfc8152)).

CWT Claims will be published soon.

1. COSE sign
   1. compact the JSON into CBOR
   1. sign and package as a COSE message
   1. ZLIB compress
   1. Base45 encode 
1. COSE verify     
   1. Base45 decode
   1. ZLIB decompress
   1. check the signature on the COSE message
   1. unpack the CBOR into JSON