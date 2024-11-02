---
layout: post
show_meta: true
title: Setting up LetsEncrypt TLS certificate on AWS
header: Setting up LetsEncrypt TLS certificate on AWS
date: 2024-10-23 00:00:00
summary: How to create a LetsEncrypt TLS certificate on AWS
categories: aws letsencrypt tls certificate-manager
author: Chee Yeo
---

In a recent project to setup a vault cluster, I required the use of a TLS certificate. On AWS, there are at least 2 ways to obtain a certificate via Certificate Manager:

* Request a public certificate
* Request a private certificate through an AWS Private CA

Since vault requires both the private key and certificate files to exist locally, we cannot use a public certificate as we cannot retrieve the private key from it. Using a Private CA is too costly as we are only using this in a development environment.

Before we can proceed, we need to have a custom domain created in Route53 with its own hosted zone.

The approach I took was to utilise [the letsencrypt API](https://certbot.eff.org/) to generate the TLS certificate:

{% highlight shell linenos %}
docker pull certbot/dns-route53:latest

docker run -it --rm --name certbot \
    --env-file env-file  \
    -v "/home/chee/.aws:/root/.aws" \
    -v "${PWD}/tls:/etc/letsencrypt" \
    -v "${PWD}/tls:/var/lib/letsencrypt" \
    certbot/dns-route53 certonly \
    -d 'teka-teka.xyz' \
    -d 'vault.teka-teka.xyz' \
    -m user@example.com \
    --agree-tos --server https://acme-v02.api.letsencrypt.org/directory
{% endhighlight %}

Certbot provides a docker image that integrates with AWS Route53. It generates a DNS entry in the hosted zone for the domain during verification. We pass in the AWS credentials using an `env-file` and map our local AWS credentials into the running container. 

Note that we need to specify each domain separately via the `-d` flag. Specifying localhost or IP addresses generates an error.

Once completed, the private key and certificate are created in the local directory `${PWD}/tls` mapped into the container:

{% highlight shell linenos %}
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/teka-teka.xyz/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/teka-teka.xyz/privkey.pem
This certificate expires on 2025-01-27.
These files will be updated when the certificate renews.
{% endhighlight %}

To use the certificates in vault setup, we can upload them to Secrets Manager as binary files:

{% highlight shell linenos %}
aws secretsmanager create-secret --name "VAULT_TLS_PRIVKEY" \
   --description "Vault Private key file" \
   --secret-binary fileb://tls/live/example.xyz/privkey.pem

aws secretsmanager create-secret --name "VAULT_TLS_CERT" \
   --description "Vault Certificate file" \
   --secret-binary fileb://tls/live/example.xyz/fullchain.pem
{% endhighlight %}

To access the certificate and private key during setup, we can invoke `aws secretsmanager get-secret-value` to download the files onto the setup ec2 instance:

{% highlight shell linenos %}
aws secretsmanager get-secret-value \
  --secret-id VAULT_TLS_PRIVKEY \
  --query 'SecretBinary' \
  --output text | base64 --decode > vault_key.pem


aws secretsmanager get-secret-value \
  --secret-id VAULT_TLS_CERT \
  --query 'SecretBinary' \
  --output text | base64 --decode > vault_ca.crt
{% endhighlight %}

{% highlight shell linenos %}
sudo chmod 0600 ${tpl_vault_storage_path}/tls/vault_ca.crt
sudo chown vault:vault ${tpl_vault_storage_path}/tls/vault_ca.crt

sudo chmod 0640 ${tpl_vault_storage_path}/tls/vault_key.pem
sudo chown vault:vault ${tpl_vault_storage_path}/tls/vault_key.pem
{% endhighlight %}

To use it in an example vault configuration file:

{% highlight shell linenos %}
storage "raft" {
  path    = "${tpl_vault_storage_path}"
  node_id = "$${INSTANCE_ID}"

  retry_join {
    auto_join_scheme = "https"
    auto_join = "provider=aws region=${tpl_aws_region} tag_key=cluster_name tag_value=vault-dev"
    leader_tls_servername = "vault.teka-teka.xyz"
    leader_client_cert_file = "${tpl_vault_storage_path}/tls/vault_ca.crt"
    leader_client_key_file = "${tpl_vault_storage_path}/tls/vault_key.pem"
  }
}
{% endhighlight %}

We can also import the certificate into Certificate Manager by going to `Import Certificate` under `Certificate Manager` and filling in the following fields:

* Certificate body => contents of cert.pem
* Certificate private key => contents of privkey.pem
* Certificate chain => contents of fullchain.pem

If successful, we should see the certificate listed on the dashboard. The screenshots below show details of the imported certificate.

![Imported Certificate in Certificate Manager](/assets/img/certbot/imported_cert.png)

We can use this imported certificate in load balancers and Cloudfront distributions but that will be in another post.

Letsencrypt TLS certificates are shortlived and needs to be renewed. For an imported certificate, we can create an Eventbridge event that triggers a lambda function that calls the certbot binary and upload the renewed certs into secrets manager. This will be explored further in a future post.

Note that DNSSEC is disabled in order for the above to work. This needs to be also done on the source domain registratr configuration if the domain is not obtained from AWS. We can use the [DNS Checker tool] to validate this.

[DNS Checker tool]: https://dnschecker.org/

### Further References

* [DNS Checker tool](https://dnschecker.org/)

* [Certbot DNS Route53](https://certbot-dns-route53.readthedocs.io/en/stable/)

* [Certbot DNS Rouet53 docker hub](https://hub.docker.com/r/certbot/dns-route53)

* [Article on LetsEncrypt on AWS](https://medium.com/w-logs/generate-standalone-ssl-certificate-with-lets-encrypt-for-aws-route-53-25a30ca3062)

