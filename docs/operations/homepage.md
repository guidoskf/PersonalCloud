# Homepage — Operational Documentation

## 1. Overview

Homepage is the centralized dashboard of PersonalCloud.

It provides a single interface for accessing the applications and infrastructure services deployed within the platform.

Homepage currently runs in the local development environment using Docker Compose.

**Current status:** Running and locally validated.

## 2. Deployment Information

Homepage is deployed as a Docker container using Docker Compose.

The deployment uses the following configuration:

- **Application:** Homepage
- **Image:** `ghcr.io/gethomepage/homepage:v2.8.0`
- **Container name:** `homepage`
- **Compose service:** `homepage`
- **Runtime:** Docker Engine
- **Deployment method:** Docker Compose
- **Restart policy:** `unless-stopped`
- **Container UID:** `1000` (PUID)
- **Container GID:** `1000` (PGID)

The image version is explicitly pinned to ensure deployment consistency and prevent unexpected updates.

The container has been verified in a healthy running state.

## 3. Network Configuration

Homepage listens on TCP port `3000` inside the container.

The service is published on the host using the following port binding:

```text
127.0.0.1:3000:3000
```

This defines:

- **Host IP:** `127.0.0.1`
- **Host port:** `3000`
- **Container port:** `3000`
- **Protocol:** TCP
- **Access scope:** Local host only
- **Compose network:** `homepage_default`

Homepage is accessible through:

`http://127.0.0.1:3000`

The service is not intentionally exposed to the local network or the Internet.

The `HOMEPAGE_ALLOWED_HOSTS` environment variable restricts accepted HTTP Host headers to:

- `localhost:3000`
- `127.0.0.1:3000`

## 4. Persistent Storage

Homepage configuration is maintained outside the container through a Docker bind mount.

**Host directory:**

`./config`

**Container directory:**

`/app/config`

**Absolute local path:**

`/home/gui/projects/PersonalCloud/compose/homepage/config`

The configuration directory contains:

- **`settings.yaml`:** General dashboard settings, including title, description, language, and appearance.
- **`services.yaml`:** Application service definitions displayed on the dashboard.
- **`bookmarks.yaml`:** External resource links and shortcuts.
- **`widgets.yaml`:** Dashboard information widgets.
- **`docker.yaml`:** Default Docker integration configuration; not currently used.
- **`kubernetes.yaml`:** Default Kubernetes integration configuration; not currently used.
- **`proxmox.yaml`:** Default Proxmox integration configuration; not currently used.
- **`custom.css`:** Custom dashboard styling.
- **`custom.js`:** Custom dashboard behavior.
- **`logs/`:** Application log files.

Application configuration is stored outside the container and survives container removal and recreation.

## 5. Dashboard Configuration

### 5.1 General Settings

The dashboard uses the following settings:

- **Title:** `PersonalCloud`
- **Description:** `PersonalCloud Administration Panel`
- **Language:** English (`en`)
- **Color scheme:** Slate

### 5.2 Services

The initial dashboard contains a group named `PersonalCloud`.

It currently includes Homepage as its first service entry.

Additional applications will be registered as they are implemented and validated.

### 5.3 Bookmarks

The dashboard contains a group named `Developer Resources`.

It currently includes:

- **GitHub:** Source code hosting and version control.
- **Docker Documentation:** Official Docker documentation.
- **Homepage Documentation:** Official Homepage documentation.

### 5.4 Information Widgets

The dashboard currently provides:

- Date and time widget.
- DuckDuckGo search widget.

The date and time widget uses the `en-GB` locale and a 24-hour clock.

### 5.5 Application Scope

Homepage is used as a centralized navigation dashboard.

It does not manage Docker containers or monitor container resources.

The following responsibilities are assigned to other PersonalCloud components:

- **Komodo:** Container and deployment management.
- **Beszel:** Resource monitoring.
- **Gatus:** Service availability monitoring.

Homepage does not currently use the Docker API or Docker socket.

## 6. Security Configuration

The following security measures are implemented:

- The container is configured with UID `1000` and GID `1000` through `PUID` and `PGID`.
- The published port is bound exclusively to `127.0.0.1`.
- Allowed HTTP hosts are restricted to `localhost:3000` and `127.0.0.1:3000`.
- The Docker Engine socket is not mounted inside the container.
- Application configuration is maintained outside the container.

The allowed-host configuration is a Host header restriction, not an authentication mechanism.

No additional user authentication, external reverse proxy, or public HTTPS endpoint has been implemented.

The current deployment is intended exclusively for local development.

Before deployment to OCI, remote access protection, authentication, HTTPS, and additional security controls must be implemented and validated.

## 7. Dependencies

Homepage currently depends on:

- Windows development host.
- WSL2 with Ubuntu.
- Docker Desktop using the WSL2 backend.
- Docker Engine.
- Docker Compose.
- Local persistent configuration files.

Docker Desktop must be running for Homepage to operate in the current development environment.

The active Docker context in Ubuntu is `default`.

## 8. Operational Validation

The following checks have been successfully completed.

### Deployment

- Docker Compose configuration validation.
- Container image download.
- Container startup.
- Container healthcheck.
- Effective container configuration inspection.
- Restart policy verification.
- Published port verification.
- Bind mount verification.

### Application Functionality

- Local HTTP access.
- Dashboard rendering.
- Service card and bookmark display.
- Date and time widget.
- DuckDuckGo search widget.
- External bookmark navigation.
- Responsive layout verification at approximately 390 pixels wide.

### Network Security

- Request using an allowed Host header returned HTTP `200`.
- Request using an unauthorized Host header returned HTTP `400`.
- Published port verified as `127.0.0.1:3000`.

### Persistence and Recovery

- Bind-mounted configuration verified.
- Container stop and start operations.
- Container restart.
- Container removal and recreation.
- Configuration preservation after recreation.
- Successful application startup after recreation.

The application recovered its configuration from the persistent host directory without requiring manual reconstruction.

## 9. Known Limitations and Future Work

The current implementation is limited to local development.

The following capabilities are not implemented:

- Public HTTPS access.
- Reverse proxy integration.
- Additional user authentication.
- Remote access protection.
- Production deployment.
- Kubernetes deployment.
- Automated application update and rollback validation.

Container monitoring and management are intentionally outside Homepage's scope.

Project-level backups have been created and successfully tested using the procedure documented in `docs/backup/backup-and-restore.md`.

This does not constitute a backup of Docker volumes, databases, or other persistent application data stored outside the project directory.

Production security, remote access, and infrastructure integration will be addressed during the OCI deployment phase.

## 10. Related Documentation

- `docs/decisions/ADR-001-homepage-deployment-architecture.md`
- `docs/runbooks/homepage-lifecycle.md`
- `docs/backup/backup-and-restore.md`
- `docs/security/`
- `ARCHITECTURE.md`
- `PROJECT_STATE.md`