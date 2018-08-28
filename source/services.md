---
title: Services
description: Setting up Engine in different environments.
---

A service in Engine is an entity that is provisioned in the Engine cloud service that your server can report performance metrics and schema versions to via an API key. The information reported by your server is processed and stored in the Engine cloud service and made accessible to you through the [Engine interface](https://engine.apollographql.com/).

## Creating a service

Services in Engine have globally unique IDs, so we usually recommend that you prefix your service ID with the name of your company or organization. To create a service, you will need to select an [account](./accounts.html) for that service to belong to. All members of the account will be able to see the service's data and settings options.

### Transfering a service

You can transfer services between any Engine account that you're a member of. To transfer a service, visit its Settings page and change the "Endpoint owner" to whichever account you'd like. If the account you'd like to transfer your service to does not appear in the options dropdown, you can contact Engine Support and we can facilitate the transfer for you.

## Different environments

You should create a different service in Engine for each of your environments. Teams using Engine will typically create 3 services: production, staging, and development. Each service has a different API key so that information can be reported to it separately, and you can set up your server to report information to the correct API key through things like environment variables. Reporting information separately allows you keep your schema versions and performance metrics in from each environment separate and individually viewable.

### API keys

API keys can be added to and removed from a service any time, and you can facilitate this on the service's Settings page. A service can have multiple API keys at once.
