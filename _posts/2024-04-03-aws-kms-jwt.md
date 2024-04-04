---
layout: post
show_meta: true
title: Using AWS KMS to sign and verify JWT
header: Using AWS KMS to sign and verify JWT
date: 2024-04-03 00:00:00
summary: Using AWS KMS to sign and verify JWT
categories: aws kms jwt api
author: Chee Yeo
---

[JWT spec]: https://jwt.io/introduction
[KMS manual key rotation]: https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html#rotate-keys-manually


In a recent project, I was looking into how to perform json web token authentication. However, I wanted to use the AWS KMS service to manage the user keys.

KMS supports both AWS-generated and customer-managed keys, which are keys created on localhost and imported into KMS. For this article, we will be creating a KMS key which is owned by AWS.

Firstly, we navigate to the KMS dashboard in the console and click on **Create a key**

This will take you to **COnfigure Key** where we use the following configuration:
* Key type -> Asymmetric
* Key usage -> Sign and verify
* Key spec -> RSA_2048
* Advanced options -> Key material origin ( KMS recommended )
* Advanced options -> Single-region key

Under **Add labels** we need to create an Alias to reference the key. I'll be using **TEST_KEY** for this article. Add an optional description or tags. 

Under **Define key administrative permissions** we need to specify the IAM users and roles who have admin control over the key i.e. to delete or rotate the keys. For the purposes of this article, I will delegate it to the AWS Profile of the development account but in production this needs to be delegated carefully. Under **Define key usage permissions**, we need to specify IAM users and roles who can use the key for signing or verification processes. Again, I delegate this to the AWS profile of the development account. IN production usage, this would be the IAM role of the application. The purposes of the IAM permissions is that it create a resource policy which is attached to the key after generation to limit who can use and administer the key for security reasons. The policy can be modified after the key is provisioned.

After the key is created, make a note of the key alias as we will be using it to generate the JWT signatures next.


#### Creating the Json Web Token

The [JWT spec] is an open standard of RFC 7519 that defines a protocol for transmitting information between two parties using a JSON object. The information can be verified as its digitially signed using a secret with HMAC algorithim, which is commonly used, or a public/private key pair ( asymmetric encryption ) using RSA / ECDSA. 

Note that the header and payload are only base64 encoded as its used to generate the eventual signature. JWT are used to verify the integrity of the claims contained within only. It's not advisable to add / store any secret information in the payload since it can be read via base64 decoding. To encrypt the payload, we have to use JWE ( Json Web Encryption ).


Using boto3, we can create a custom python function to generate the JWT:

{% highlight python %}
def create_jwt(key_id: str) -> str:
    issued_at = int(time.time())
    # expires 1 hr from now
    expired_at = 3600 + issued_at

    header = {
        "alg": "RS256",
        "typ": "JWT"
    }

    payload = {
        "iat": issued_at,
        "exp": expired_at
    }

    token_components = {
        "header": base64.urlsafe_b64encode(json.dumps(header).encode()).decode().rstrip("="),
        "payload": base64.urlsafe_b64encode(json.dumps(payload).encode()).decode().rstrip("=")
    }

    message = f'{token_components["header"]}.{token_components["payload"]}'

    client = boto3.client('kms', region_name='eu-west-1')
    signature = client.sign(
        KeyId=key_id,
        Message=message,
        SigningAlgorithm="RSASSA_PKCS1_V1_5_SHA_256",
        MessageType="RAW"
    )["Signature"]

    token_components["signature"] = base64.urlsafe_b64encode(signature).decode().rstrip("=")

    return f'{token_components["header"]}.{token_components["payload"]}.{token_components["signature"]}'
{% endhighlight %}

The function takes in **key_id** parameter which will be the alias of the KMS key to use. 

Within the function, we define the header and payload as dicts, as defined within the RFC. These are base64 encoded and then converted to string to remove any additional '=' characters. Next, we instantiate a KMS client object and pass in the key alias and the message to sign which is the header and payload concatenated via a single dot as per the RFC. We invoke the **sign** method with the key alias, the mssage, the signing algorithm and the message type. Note that since we specified **RS256** in the header dict, we need to select a signing algorithm of **RSASSA_PKCS1_V1_5_SHA_256** as stated in the RFC:

> It is RECOMMENDED that implementations also support RSASSA-PKCS1-v1_5 with the SHA-256 hash algorithm ("RS256")

The **MessageType** must also be set to **RAW** since we are passing in the actual base64 encoded message.

After we created the signature, we apply base64 encoding to it and concatenate it with the header and payload as a single string and returns it to the client.


#### Verifying the Json Web Token

After obtaining the JWT, a client would send it to the target endpoint usually in the request header as **Authorization: Bearer <token>**

The server can check for the presence of the token on protected routes. If present, the validity of the token can be verified in the following ways:

* Check that the token has not expired via the **exp** field in the payload body

* Check that the signature in the token matches the signature generated from the KMS keys.

The KMS client provides a method **verify** that we can use to verify the signature:

{% highlight python %}
def verify_jwt(jwt_str: str, key_id: str) -> bool:
    jwt_parts = jwt_str.split('.')
    signed_str = '.'.join(jwt_parts[0:2]).encode()
    
    # NOTE: The == is needed as the orig signature has it before encoded into bytes..
    signature = base64.urlsafe_b64decode(jwt_parts[2] + '==')

    client = boto3.client('kms', region_name='eu-west-1')

    resp = client.verify(
        KeyId=key_id,
        Message=signed_str,
        MessageType='RAW',
        SigningAlgorithm="RSASSA_PKCS1_V1_5_SHA_256",
        Signature=signature
    )

    return resp['SignatureValid']
{% endhighlight %}

The JWT is passed into the function together with the KMS key alias. We obtain the original message that was used to create the signature by splitting the JWT on the dot. We join the header and payload into a single byte stream. We apply base64 decoding to the signature. Since our original create function removed the additional '=' characters generated by the sign function, we need to add it back to the decoded string else it will fail with **Incorrect padding**

Next we invoke the KMS client and invoke the **verify** method with the key alias, the message string, the signature. We also set the message type to RAW and the signing algorithm to match the signing process.

The response is a boolean value indicating if the signature is valid or not. The server can return an appropriate response based on this boolean value.


Using KMS to create and manage the keys allow for this process to be scalable and serverless. We don't have to manage the encryption keys manually. This is also highly secure as KMS don't export any key private key material. The fine-grained resource policy also means we can enforce separation and be able to track any key access via CloudTrail service for auditing.


#### On KMS key rotation

While KMS supports key rotation, it only applies to symmetric keys and AWS managed keys. For customer-managed keys (CMK), we can apply our own key rotation as follows:

* Create a new KMS key with a unique alias value
* Update the target alias value with the key ARN of the new key. This will disassociate the old key from the alias.
* Check that the new key has the alias applied.

{% highlight bash %}
aws kms list-aliases --query 'Aliases[?AliasName==`alias/TestKey`]'

aws kms update-alias --alias-name alias/TestKey --target-key-id <key arn>

aws kms list-aliases --query 'Aliases[?AliasName==`alias/TestKey`]'
{% endhighlight %}

Note that if the KMS key is used to encrypt / decrypt files, we need the old key to be around in order for the decryption to work. This is detailed in [KMS manual key rotation]


Hope that this post helps in understanding how we can use KMS in applications.

H4PPY H4CK1NG !