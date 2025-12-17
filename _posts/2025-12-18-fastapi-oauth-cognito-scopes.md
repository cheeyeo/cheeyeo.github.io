---
layout: post
show_meta: true
title: Securing FastAPI with OAuth and custom user scopes using AWS Cognito
header: Securing FastAPI with OAuth and custom user scopes using AWS Cognito
date: 2025-12-18 00:00:00
summary: How to secure a FastAPI application using with custom user scopes using AWS Cognito
categories: python fastapi web oauth2 aws cognito
author: Chee Yeo
---

In a previous post, I showed an example of integrating FastAPI with Cognito as an OAuth provider. Another advanced usage is to introduce scopes into the OAuth flow to refine the authorization of the API.

Scopes are an additional field within the access token that can be used to restrict access to API resources. The default scope returned by Cognito is `aws.cognito.signin.user.admin me` which allows the user to administer their own profile such as updating username etc. 

To add custom scopes to an access token, the recommended approach is to create a Resource server to define the custom scopes and a custom domain with TLS enabled. The application would need to call the custom domain at the Token Endpoint of `/oauth/endpoint` to retrieve the access token with the custom scopes. Usage of the cognito API via cognito-idp boto3 client will not allow retrieval of the custom scopes.

Another approach is to create a Lambda trigger which would allow you to add the custom scopes into the access token before its returned from Cognito. This lambda function is known as a PreToken generation Lambda. 

TODO: screenshot of how to create the lambda and linked it to user pool in cognito


TODO: Lambda code in python showing custom scopes.


