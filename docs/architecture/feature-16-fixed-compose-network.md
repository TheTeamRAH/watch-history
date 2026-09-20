# Feature 16: Fixed Compose Network

## Status

In progress. The approach was reviewed with the user on 20 September 2026.

## Goal

Give the application web container a stable source address when it calls a Home Assistant instance running in Docker host-network mode. This lets the Home Assistant reverse proxy apply a narrow IP allowlist without depending on Docker's dynamically assigned container addresses.

## Context

The Home Assistant import was rejected with HTTP 403 after an IP allowlist was introduced. The import worker calls the web service, and the web service makes the authenticated Home Assistant request. Therefore, only the `web` container needs a stable egress address for this integration.

## Decisions

- The application Compose network uses the fixed subnet `172.37.0.0/24`.
- The network gateway is `172.37.0.1` and `web` is assigned `172.37.0.5`.
- `worker` and `db` retain dynamic addresses; neither needs direct Home Assistant access.
- Home Assistant remains in host-network mode and is not added to the application bridge.
- `web` receives a Linux Docker host-gateway alias at `host.docker.internal` for configurations that need to reach Home Assistant through the host.
- Home Assistant or its reverse proxy must allowlist `172.37.0.5` after deployment, provided its denial log confirms that address is the request source.

## Scope

- Configure the existing default Compose bridge network with the approved subnet and stable web address.
- Document the deployment and allowlist workflow.
- Keep existing internal service names (`web` and `db`) working on the default network.

## Non-goals

- Changing Home Assistant's Docker configuration or host-network mode.
- Changing the ignored, environment-specific `configs/home-assistant.yaml` file.
- Broadly allowlisting Docker bridge subnets.
- Altering Home Assistant token handling or public error responses.

## Verification

On the active Docker host, after deploying the Compose update:

1. Run `docker compose config` and confirm the default network uses `172.37.0.0/24` with gateway `172.37.0.1`, and `web` resolves to `172.37.0.5`.
2. Run `docker compose up -d --force-recreate` to apply the network change.
3. Confirm `docker compose ps` shows `web`, `worker`, and `db` healthy/running as applicable.
4. Update the Home Assistant reverse-proxy allowlist with `172.37.0.10` and confirm its denial log reports that source address.
5. Trigger a Home Assistant import and confirm it completes.
