# claw_docker_cli

Public wrapper image for running OpenClaw Gateway with the Docker CLI available inside the container.

The image is built from the official OpenClaw image and copies in the Docker CLI binary, so OpenClaw can inspect/control Docker when the host Docker socket is mounted.

## Image

```yaml
image: ghcr.io/oywino/openclaw-gateway-dockercli:<openclaw-version>
```

Example:

```yaml
image: ghcr.io/oywino/openclaw-gateway-dockercli:2026.6.9
```

## What the GitHub Action does

The workflow in `.github/workflows/auto-build.yml` runs on a schedule and can also be started manually.

It:

1. Reads the latest OpenClaw release tag from `openclaw/openclaw`.
2. Builds this wrapper image with:

   ```dockerfile
   FROM ghcr.io/openclaw/openclaw:${OPENCLAW_TAG}
   ```

3. Copies the Docker CLI from `docker:27-cli` into the OpenClaw image.
4. Publishes the result to:

   ```text
   ghcr.io/oywino/openclaw-gateway-dockercli:<openclaw-version>
   ```

The intent is that the wrapper tag mirrors the upstream OpenClaw version tag.

## Stable releases only

For Øyvind's deployment, only stable OpenClaw releases should be used.

Ignore prerelease tags and non-default image variants.

Prerelease/channel identifiers to ignore include:

- `beta`
- `alpha`
- `rc`
- `nightly`
- `dev`
- other prerelease identifiers

Non-default image variants to ignore for this deployment include:

- `browser`

The local deployment should be updated manually by editing the compose image line after a stable default tag has been confirmed.

## Local docker-compose usage

Example service:

```yaml
services:
  openclaw-gateway:
    image: ghcr.io/oywino/openclaw-gateway-dockercli:2026.6.9
    restart: unless-stopped
    ports:
      - "18789:18789"
    environment:
      - TZ=Europe/Oslo
    volumes:
      - /share/Public/openclaw/config:/home/node/.openclaw
      - /share/Public/openclaw/workspace:/home/node/.openclaw/workspace
      - /var/run/docker.sock:/var/run/docker.sock
      - /share/Drive_D:/mnt/drive_d:ro
      - /share/Ubuntu_SB:/mnt/ubuntu_sb:ro
```

## Local OpenClaw cron reminder/check

A local OpenClaw cron job is used as a reminder/checker. It does **not** update the container automatically.

Purpose:

- Check whether a newer **stable** OpenClaw version exists.
- Confirm whether the matching custom wrapper tag exists at:

  ```text
  ghcr.io/oywino/openclaw-gateway-dockercli:<version>
  ```

- Tell Øyvind the exact compose line to use manually.
- Avoid recommending beta/prerelease/dev tags or non-default variants such as browser.

Current confirmed deployment baseline:

```yaml
image: ghcr.io/oywino/openclaw-gateway-dockercli:2026.6.9
```

### Cron job config

This is the redacted/reusable cron configuration. Delivery target details are intentionally omitted from this public README.

```json
{
  "name": "openclaw: stable docker image update check weekly",
  "description": "Weekly stable-only OpenClaw Docker image tag check; reports newer tag for manual compose update.",
  "enabled": true,
  "schedule": {
    "kind": "cron",
    "expr": "17 3 * * 0",
    "tz": "Europe/Oslo"
  },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "timeoutSeconds": 600,
    "message": "Weekly reminder/check: check whether a newer STABLE default OpenClaw Gateway Docker image version is available for Øyvind to manually put into docker-compose. Current confirmed compose baseline from Øyvind is `ghcr.io/oywino/openclaw-gateway-dockercli:2026.6.9`. IMPORTANT: ignore prereleases such as beta/alpha/rc/nightly/dev, and ignore non-default image variants such as browser; only stable default releases matter. Do not update anything automatically.\n\nTask:\n1) Determine the currently configured/running image tag if possible. If Docker socket access is permission-denied, do not treat that as failure; use the confirmed baseline `2026.6.9`.\n2) Check official OpenClaw stable default release/container tags from authoritative sources, preferably GitHub releases/packages for `openclaw/openclaw` and/or `ghcr.io/openclaw/openclaw`. Also check whether `ghcr.io/oywino/openclaw-gateway-dockercli:<tag>` has a matching stable default tag before recommending that exact custom image tag.\n3) Compare semantic/calendar-style version tags. Exclude prereleases and non-default image variants. If a newer stable default version than the current baseline exists, send a short summary with the current tag, newest stable default OpenClaw version found, and exact compose image line to use. Prefer `ghcr.io/oywino/openclaw-gateway-dockercli:<version>` only if that custom tag exists; otherwise say the custom tag was not found and identify the official stable default tag/source.\n4) If no newer stable default version exists, send a brief OK summary with current tag and newest stable default tag checked.\n5) If sources are unreachable or ambiguous, say that clearly and do not recommend a beta/prerelease or non-default variant.\n\nKeep output concise. This is only a notification/check job, not an updater."
  },
  "delivery": {
    "mode": "announce",
    "channel": "telegram",
    "to": "<redacted>",
    "accountId": "default"
  }
}
```

### Expected notification behavior

If a newer stable wrapper image exists, the notification should include something like:

```text
New stable OpenClaw wrapper image available.
Current: ghcr.io/oywino/openclaw-gateway-dockercli:2026.6.9
Newest stable: <version>
Use this compose line:
image: ghcr.io/oywino/openclaw-gateway-dockercli:<version>
```

If there is no newer stable version, it should send a short OK message instead.
