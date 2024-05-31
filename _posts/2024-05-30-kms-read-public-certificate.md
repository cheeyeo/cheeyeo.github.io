---
layout: post
show_meta: true
title: Extract Public Key from KMS 
header: Notes on how to extract public key from KMS key
date: 2024-05-30 00:00:00
summary: Notes on how to extract public key from KMS key
categories: aws kms go-lang
author: Chee Yeo
---

[GetPublicKey Documentation]: https://docs.aws.amazon.com/kms/latest/APIReference/API_GetPublicKey.html
[x.509 ParsePKIXPublicKey]: https://pkg.go.dev/crypto/x509#ParsePKIXPublicKey

In a recent project, I was exploring how to perform encryption / decryption using asymmetric keys created in AWS KMS. I discovered that there are three main ways of retrieving the public key:

* via the console
* via the AWS CLI
* via the SDK

The first method is the easiest. From AWS console, when you navigate to the key in AWS KMS, you should see a tab under details which allows you to download the public key in PEM format.

For the second method of using the aws cli we can invoke the `get-public-key` sub command:
{% highlight shell %}
aws kms get-public-key \
  --key-id $KEY_ID \
  --output text --query PublicKey | base64 --decode > publickey.der
{% endhighlight %}

The $KEY_ID could be either the full key ARN or the alias. I find that creating an alias is easier than using the full ARN. Note that the output is base64 encoded only when using the AWS CLI so we need to apply base64 decode before saving it.

Using the AWS SDK, we can invoke the `GetPublicCertificate` API endpoint to retrieve the public key. Below is an example using the AWS go-lang SDK:

{% highlight go %}
func GetPublicKey(client kmsiface.KMSAPI, keyID string) ([]byte, error) {
	// Gets the PublicKey from an Asymmetric key
	output, err := client.GetPublicKey(&kms.GetPublicKeyInput{
		KeyId: aws.String(keyID),
	})

	if err != nil {
		return nil, err
	}

	// output is GetPublicKeyOutput
	// Get the public key as byte array
	return output.PublicKey, nil
}
{% endhighlight %}

As per the [GetPublicKey Documentation], the public key is returned as a DER-encoded X.509 public key, also known as SubjectPublicKeyInfo. We need to parse the X.509 Public key in order to obtain the public key required for the downstream algorithm to use. 

In go-lang we can use [x.509 ParsePKIXPublicKey] to parse the DER file. The function returns parsed public key in different formats, which include RSA, DSA, ECDSA and ED25519. For my use case, I only needed the public key to be in RSA format:
{% highlight go %}
func ConvertDERToRSA(data []byte) (*rsa.PublicKey, error) {
	pub, err := x509.ParsePKIXPublicKey(data)
	if err != nil {
		return nil, err
	}

	switch pub := pub.(type) {
	case *rsa.PublicKey:
		fmt.Println("Public key is of type RSA")
		return pub, nil
	default:
		return nil, errors.New("invalid RSA Public key")
	}
}
{% endhighlight %}

Generally, one should not have to download the public key as calling encrypt / decrypt via KMS API automatically uses the public key on the HSM device in KMS. The following are some situations when this might be useful:

* When access to KMS API is not practical such as the use of AWS credentials on distributed IOT devices or on devices you don't control. The public key can be downloaded and stored separately for client usage.

* When files can be large as RSA has a limit of 190 bytes. We can use block cipher algorithms such as AES to generate a private key to encrypt the file and then use the public key to encrypt the private key for decryption later.

Hope this helps. H4PPY H4CK1NG