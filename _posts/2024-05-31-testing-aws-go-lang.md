---
layout: post
show_meta: true
title: Test AWS services using go-lang SDK
header: Notes on how to test AWS services when using go-lang SDK
date: 2024-05-31 00:00:00
summary: Notes on how to test AWS services when using go-lang SDK
categories: aws go-lang testing
author: Chee Yeo
---

[AWS GO SDK]: https://docs.aws.amazon.com/sdk-for-go/api/
[KMSIFACE]: https://docs.aws.amazon.com/sdk-for-go/api/service/kms/kmsiface/

In a recent project using the [AWS GO SDK], I was trying to figure out how to test custom code which uses the SDK. These are in the form of a client API call which uses a service client.

For example, below is an example of a function that calls the KMS Encrypt API:

{% highlight go %}
func EncryptKey(client *kms.KMS, keyId string, source []byte, target string) error {
	// Encrypt the data
	result, err := client.Encrypt(&kms.EncryptInput{
		KeyId:               aws.String(keyId),
		Plaintext:           source,
		EncryptionAlgorithm: aws.String("RSAES_OAEP_SHA_256"),
	})

	if err != nil {
		return err
	}

	err = os.WriteFile(target, result.CiphertextBlob, 0644)
	if err != nil {
		return err
	}

	return nil
}
{% endhighlight %}

It accepts as parameters a kms client, the key ID, the source file to encrypt and the target path to save the encrypted file.

To test the above, we could mock the client parameter and stub out the `Encrypt` function call. The AWS SDK provides interfaces to achieve the above for each of the service supported. For example, we could use [KMSIFACE] to mock the service client.

Firstly, we need to rewrite the function above and replace the client with `kmsiface.KMSAPI`:

{% highlight go %}
func EncryptKey(client kmsiface.KMSAPI, keyId string, source []byte, target string) error {
	// Encrypt the data
	result, err := client.Encrypt(&kms.EncryptInput{
		KeyId:               aws.String(keyId),
		Plaintext:           source,
		EncryptionAlgorithm: aws.String("RSAES_OAEP_SHA_256"),
	})

	if err != nil {
		return err
	}

	err = os.WriteFile(target, result.CiphertextBlob, 0644)
	if err != nil {
		return err
	}

	return nil
}
{% endhighlight %}

The `kmsiface.KMSAPI` interface supports all the same functions as the `kms.KMS` client.

Next, we create a mock client that implements the interface. I added an additional field to the mock client struct which accepts an optional error. If it exists, the mock function call will return the error only, to simulate an actual error occuring.

The mock client would need to implement each function of the service client you want to test. For the example above, we implement a stub version of the `Encrypt` function as follows:

{% highlight go %}
type mockKMSClient struct {
	kmsiface.KMSAPI
	raiseErr error
}

func (m *mockKMSClient) Encrypt(*kms.EncryptInput) (*kms.EncryptOutput, error) {
	if m.raiseErr != nil {
		return nil, m.raiseErr
	}

	output := &kms.EncryptOutput{
		CiphertextBlob: []byte("ciphertext"),
	}
	return output, nil
}
{% endhighlight %}

Using the mock client, if an error exists, it would return nil and the error. If not, it returns the encrypt output with a stub value.

To use it in a unit test:
{% highlight go %}

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestEncryptKey(t *testing.T) {
	mockSvc := &mockKMSClient{
		raiseErr: nil,
	}

	err := EncryptKey(mockSvc, "XXX", []byte("this is some text"), "/tmp/target")
	_, err2 := os.Stat("/tmp/target")
	assert.Nil(t, err)
	assert.Nil(t, err2)
	os.Remove("/tmp/target")
}

func TestEncryptKeyError(t *testing.T) {
	mockSvc := &mockKMSClient{
		raiseErr: errors.New(kms.ErrCodeNotFoundException),
	}

	err := EncryptKey(mockSvc, "XXX", []byte("this is some text"), "/tmp/target")
	_, err2 := os.Stat("/tmp/target")
	assert.NotNil(t, err)
	assert.NotNil(t, err2)
	os.Remove("/tmp/target")
}
{% endhighlight %}

The unit test above runs two different test cases. In the unsuccessful case, we pass an error of not found into the mock client and assert that the target file doesn't exist and errors are returned. In the successful case, we assert that the target file exists and no errors are thrown.

While the above might be verbose, using the `kmsiface` interface client means we have a single unified client for both the test and production code which makes it easier to only stub out the functions we want to test. 

Hope it helps!!!