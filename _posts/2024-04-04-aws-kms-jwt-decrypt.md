---
layout: post
show_meta: true
title: Using AWS KMS to decrypt JWT payload
header: Using AWS KMS to decrypt JWT payload
date: 2024-04-04 00:00:00
summary: Using AWS KMS to decrypt JWT
categories: aws kms jwt api
author: Chee Yeo
---

[KMS get public key]: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/kms/client/get_public_key.html


In a previous post, I detailed how we can use an AWS KMS key to sign and verify JWT. 

We can also use the public key to decrypt the token content and retrieve the sent payload.

{% highlight python %}
def decrypt_jwt(signature: str, key_id: str) -> tuple[dict, dict]:
    client = boto3.client('kms', region_name='eu-west-1')
    resp = client.get_public_key(
        KeyId=key_id
    )['PublicKey']

    # The public key is returned in DER encoded X.509 public key
    # only when using CLI or http api its returned base64 encoded
    key = serialization.load_der_public_key(resp)

    header_data = jwt.get_unverified_header(signature)
    payload_data = jwt.decode(signature, key, algorithms=[header_data['alg']])

    return header_data, payload_data
{% endhighlight %}

The function above takes as input the JWT signature and the KMS key alias that was used to create the signature.

We instantiate a KMS client and invoke the **get_public_key** method which accepts the key alias.

Note that as documented in [KMS get public key], if we are using either the CLI or HTTP API, the public key is returned in base64 encoded format. We can see this from within the console itself where we can download the public key of an asymmetric keypair. 

When using the boto3 library, the returned public key is in DER encoded format. We need to invoke **serialization.load_der_public_key** to obtain the public key in bytes. 

We pass the returned public key to the **decode** function of the **Py-jwt** library which decrypts the payload using the public key.

Note that this is only applicable if we only have the JWT signature. If we have the entire JWT which includes the header and payload, we can simply perform base64 decoding on the header and payload from the token itself.

Hope it helps. 

H4PPY H4CK1NG