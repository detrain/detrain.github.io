---
title: OpenClaw Containerization
date: 2026-02-19 21:24:00 -0600 
categories: [OpenClaw]
tags: [openclaw]     # TAG names should always be lowercase
---
This post addresses how to containerize multiple OpenClaw instances using the recent Docker additions to [OpenClaw](https://github.com/openclaw/openclaw). Why? It enables isolated setups for...

- Multiple users have their own instance
- Keep a successful instance protected from bad prompting / behavior
- Experimentation to let yourself go wild and not care (as much) what OpenClaw builds (or destroys...)

**The sky is the limit!**

> **Note:** Setup installed inside macOS
{: .prompt-info }

> **Forewarning:** There may be a better way to do this, but I did not locate a solution at the time of working this out. And so hopefully this helps others get something that works.
{: .prompt-warning}

## Prerequisites
- Docker (and Docker Compose -- comes with Docker Desktop)
- Node.js
- OpenClaw
- git

## Plan
1. Create OpenClaw configuration(s) on your physical hardware
2. Migrate configuration(s) to container(s)
3. Profit

## Create Configuration(s)
As of writing this I could not get OpenAI OAuth to authenticate inside the container. Therefore, I created the unique instances in the following way.

If you have a previous OpenClaw, save it off. Or if you just need to containerize this instance, skip down to the next bit.
```bash
mv ~/.openclaw ~/.openclaw-bk
```

For each instance you would like to create, go through the initial OpenClaw onboarding and authenticate to your provider. In between configurations you may need to shutdown the gateway.
```bash
openclaw onboard
```
We now have the default configuration directory `~/.openclaw`. Save this off elsewhere.

Example layout:
```bash
>> tree configs
── configs
   ├── alice-openclaw
   ├── bob-openclaw
   └── eve-openclaw
```

## Containerization
Ensure you have [OpenClaw](https://github.com/openclaw/openclaw) cloned. Ensure it has the following files:
```bash
.env
Dockerfile
docker-compose.yml
docker-setup.sh
```

Run `docker-setup.sh` to create the Docker image and then abandon initial setup.
```bash
# Make sure you are in the cloned openclaw repository
./docker-setup.sh
```

Each instance needs its own `.env` to specify its unique properties. I rolled the ports in pairs, this could have been scripted, but here we are in the age of LLMs with some good old fashioned key pressin'.

For Alice's .env
```bash
# .env-alice
OPENCLAW_CONFIG_DIR=/Users/user/configs/alice-openclaw
OPENCLAW_WORKSPACE_DIR=/Users/user/configs/alice-openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=<REDACTED>
OPENCLAW_IMAGE=openclaw:local
OPENCLAW_EXTRA_MOUNTS=
OPENCLAW_HOME_VOLUME=
OPENCLAW_DOCKER_APT_PACKAGES=
```

For Bob's .env
```bash
# .env-bob
OPENCLAW_CONFIG_DIR=/Users/user/configs/bob-openclaw
OPENCLAW_WORKSPACE_DIR=/Users/user/configs/bob-openclaw/workspace
OPENCLAW_GATEWAY_PORT=18791
OPENCLAW_BRIDGE_PORT=18792
... SNIPPED ...
```

And so on...

## Profit
With the docker configuration setup complete, we move onto starting the containers.
```bash
## Ensure you are in the openclaw repository
# For Alice
docker compose --env-file <path/to/.env-alice> -p alice up -d
# For Bob
docker compose --env-file <path/to/.env-bob> -p bob up -d
# For Eve
docker compose --env-file <path/to/.env-eve> -p eve up -d
```
All of the containers associated with the seperate instance will be running. Confirm with:
```bash
docker ps
```
Note the unique published ports per container. You will need to forward these approriately via tunnels to your respective users.

To test a particular instances gateway, fetch the dashboard.
```bash
docker compose -p alice run --rm openclaw-cli dashboard
```

## Device Token Mismatch Fix
> `failed: gateway closed (1008): unauthorized: device token mismatch (rotate/reissue device token)`
{: .prompt-warning}

```bash
# Get the valid URL w/ token
docker compose -p alice run --rm openclaw-cli dashboard
# Note the URL will be that of the local port inside the container, to access externally use the published port

# Enumerate device list
docker compose -p alice run --rm openclaw-cli devices list

# Approve the request
docker compose -p alice run --rm openclaw-cli devices approve <requestId>
```
