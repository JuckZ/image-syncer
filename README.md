# image-syncer

A lightweight GitHub Actions template for synchronizing container images from public or private registries to your own registry by using [AliyunContainerService/image-syncer](https://github.com/AliyunContainerService/image-syncer).

It is designed for three common scenarios:

- Mirror frequently used images to an internal registry.
- Keep Kubernetes, Docker Hub, Quay, or other upstream images available in restricted networks.
- Maintain a simple, auditable YAML mapping of source images to target images.

## How it works

The workflow builds a runtime `config.yaml` from two parts:

1. **Registry credentials and optional image-syncer settings** stored in a GitHub Actions secret such as `CONFIG`.
2. **Image mappings** committed in [`imagelist.yaml`](./imagelist.yaml).

The generated config is passed to `image-syncer` and is not committed back to the repository.

## Quick start

1. Fork this repository.
2. Copy [`example/config.yaml`](./example/config.yaml), replace the placeholder registry and credentials, and store the file content as a GitHub Actions secret named `CONFIG`:
   - Repository **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.
3. Edit [`imagelist.yaml`](./imagelist.yaml) and replace the example mappings with the images you want to sync.
4. Run **Actions** → **Sync container images** → **Run workflow**.

The workflow also runs automatically when `imagelist.yaml`, `example/config.yaml`, or the workflow file changes on the `main` or `master` branch.

## Configuration secret

The secret should contain the authentication block for every target registry that needs credentials. Example:

```yaml
auth:
  registry.example.com:
    username: your-username
    password: your-password
```

You can add supported `image-syncer` options to the same secret when needed, for example retry or concurrency settings:

```yaml
auth:
  registry.example.com:
    username: your-username
    password: your-password
retries: 3
proc: 6
```

> Do not commit real credentials. Keep them in GitHub Actions secrets only.

## Image mapping format

Add active image mappings under the `images` key:

```yaml
images:
  docker.io/library/nginx:1.27-alpine: registry.example.com/mirror/nginx:1.27-alpine
  quay.io/prometheus/node-exporter:v1.8.2: registry.example.com/mirror/prometheus-node-exporter:v1.8.2
```

Rules of thumb:

- The left side is the source image.
- The right side is the destination image.
- Always include explicit tags for reproducible syncs.
- Use `historyRecord` only as an optional archive for mappings you no longer sync.

## Manual workflow options

When starting the workflow manually, you can override these defaults:

| Input | Default | Purpose |
| --- | --- | --- |
| `image_list_path` | `imagelist.yaml` | Use another mapping file without editing the workflow. |
| `config_secret_name` | `CONFIG` | Use a different GitHub Actions secret for credentials/config. |
| `image_syncer_version` | `v1.5.5` | Test or pin another `image-syncer` release. |
| `dry_run` | `false` | Validate input composition and print generated top-level keys without syncing images. |

## Repository layout

```text
.github/workflows/sync-images.yml  # GitHub Actions workflow
example/config.yaml                # Safe credential/config template
imagelist.yaml                     # Source-to-target image mappings
```

## Troubleshooting

### `Secret 'CONFIG' is empty or missing`

Create the repository secret named `CONFIG`, or run the workflow manually and set `config_secret_name` to the secret you want to use.

### `Image list 'imagelist.yaml' does not exist`

Confirm the mapping file exists at the path configured by `image_list_path`.

### Authentication or push failures

Check that:

- The target registry hostname in `auth` exactly matches the hostname in the destination image.
- The configured account can create repositories or push tags in the destination namespace.
- The destination image name follows your registry provider's naming rules.

### Need to test without pushing images

Run the workflow manually with `dry_run` set to `true`. This checks that the secret and image list can be combined into a YAML config without invoking `image-syncer`.

## Maintenance tips

- Keep `image_syncer_version` pinned for reproducibility, then update it intentionally.
- Prefer immutable version tags over `latest` for upstream images.
- Move old mappings to `historyRecord` if you need an audit trail; otherwise remove them to keep the file short.
