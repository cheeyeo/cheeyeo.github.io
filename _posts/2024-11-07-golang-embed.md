---
layout: post
show_meta: true
title: Using golang embed to build web applications
header: Using golang embed to build web applications
date: 2024-11-07 00:00:00
summary: How to use golang embed directive to store UI and database assets
categories: golang embed reactjs websocket
author: Chee Yeo
---

In a recent personal project, I had a requirement to create a small web application that uses websocket which could be portable across different devices. The traditional structure I would take would generally involve using docker compose with the UI, database and backend services running in different containers. While this works for a traditional webapp stack, the requirements I had was more constrained. The application must be packaged into a single binary and be able to run across different platforms. In addition, it must also support the websocket protocol. 

Eventually, I managed to create a single binary using golang and the embed directive which stores the migration files and UI asset within a single binary which are accessed when run.

[SHOW SCREENSHOT]

This is inspired by the Vault binary which does something similar by embedding its UI contents into the CLI.

### Process

describe the app and the gorilla websocket lib

describe how you solved the issue of multi rooms and converting the gorilla broadcast example

describe the reactjs app and the libs used; how you pass the room id as query parameters


### Building

how the embed directive works

what was embedded and how it was read

