Garage [![status-badge](https://woodpecker.deuxfleurs.fr/api/badges/1/status.svg)](https://woodpecker.deuxfleurs.fr/repos/1)
===

<p align="center" style="text-align:center;">
	<a href="https://garagehq.deuxfleurs.fr">
	<img alt="Garage logo" src="https://garagehq.deuxfleurs.fr/img/logo.svg" height="200" />
	</a>
</p>

<p align="center" style="text-align:center;">
	[ <strong><a href="https://garagehq.deuxfleurs.fr/">Website and documentation</a></strong>
	| <a href="https://garagehq.deuxfleurs.fr/_releases.html">Binary releases</a>
	| <a href="https://git.deuxfleurs.fr/Deuxfleurs/garage">Git repository</a>
	| <a href="https://matrix.to/#/%23garage:deuxfleurs.fr">Matrix channel</a>
	]
</p>

Garage is an S3-compatible distributed object storage service
designed for self-hosting at a small-to-medium scale.

Garage is designed for storage clusters composed of nodes running
at different physical locations,
in order to easily provide a storage service that replicates data at these different
locations and stays available even when some servers are unreachable.
Garage also focuses on being lightweight, easy to operate, and highly resilient to
machine failures.

Garage is built by [Deuxfleurs](https://deuxfleurs.fr),
an experimental small-scale self hosted service provider,
which has been using it in production since its first release in 2020.

Learn more on our dedicated documentation pages:

- [Goals and use cases](https://garagehq.deuxfleurs.fr/documentation/design/goals/)
- [Features](https://garagehq.deuxfleurs.fr/documentation/reference-manual/features/)
- [Quick start](https://garagehq.deuxfleurs.fr/documentation/quick-start/)

Garage is entirely free software released under the terms of the AGPLv3.

## This checkout's Docker Compose deployment

The [Compose configuration](./docker-compose.yml) runs Garage and
[Garage Web UI](https://github.com/khairul169/garage-webui) on the existing
external `dokploy-network`. The WebUI can manage buckets and S3 access keys,
and browse, upload, download, and delete objects. It is a third-party
administration tool, not an isolated multi-user file-sharing portal: treat
everyone with a WebUI login as a Garage administrator.

### Environment and HTTPS routing

1. Set the variables from [.env.example](./.env.example) in Dokploy's
   environment editor. For local Compose use, copy it to `.env` (git-ignored).
   Keep your existing RPC secret and admin token; do not rotate them just to
   add the UI. The UI receives the same admin token as Garage through
   `API_ADMIN_KEY`, because the mounted [configuration](./config/garage.toml)
   does not contain the token.
2. Generate a WebUI login with `htpasswd -nBC 10 admin` and enter a strong,
   unique password at the prompt. Set `GARAGE_WEBUI_AUTH` to the resulting
   `admin:<bcrypt-hash>` value. In a local `.env` file, single-quote the entire
   value so Compose preserves the hash's `$` characters. Do not paste the
   hash into Compose YAML, where literal `$` characters need `$$` escaping.
   The login password is separate from both the Garage admin token and S3
   secret access keys. Compose refuses to deploy the UI without a non-empty
   admin token and login setting.
3. Configure these domains in Dokploy, with HTTPS enabled and path `/`:

   | Domain | Compose service | Container port |
   |--------|-----------------|----------------|
   | `garage.yunfeizhao.com` | `garage-webui` | `3909` |
   | `s3.yunfeizhao.com` | `garage` | `3900` |

   If `garage.yunfeizhao.com` currently targets Garage itself, change its
   service and port to the WebUI. Do not expose Garage's admin port `3903`.
   No host port publication is needed: Dokploy's proxy reaches the containers
   over `dokploy-network`. Prefer restricting the UI to a VPN or an IP
   allowlist as an additional layer of protection.
4. Redeploy in Dokploy, then open `https://garage.yunfeizhao.com` and log in.
   For local Compose validation use `docker compose config --quiet`; for
   deployment use `docker compose up -d`. Local deployment also requires the
   external network and storage mount paths to exist.

### Buckets, credentials, and files

For a new installation, first assign the Garage node a storage capacity and
apply the cluster layout (through the UI's layout management or Garage CLI).
An already initialized cluster does not need to be initialized again.

In the UI, create a bucket and an access key, then grant that key **read** and
**write** permissions on the bucket. Save the generated **access key ID** and
**secret access key** in a password manager, not in Git. Use a separate,
least-privilege key for each application; grant bucket-creation/owner
permissions only when needed. Use the integrated bucket/object browser to
upload a small test file, list it, and download it to verify its contents.

For external S3 clients, use:

- Endpoint: `https://s3.yunfeizhao.com`
- Region: `garage`
- Credentials: the generated S3 access key ID and secret access key
- Addressing: path-style (unless wildcard bucket DNS/TLS has been configured)

The UI's backend accesses S3 internally at `http://garage:3900` and proxies
object operations, so its object browser does not require browser access to
that internal hostname or S3 CORS changes. A separate browser application
accessing the public S3 endpoint directly does require bucket CORS rules.
Never give an S3 client the Garage admin token.

The WebUI image is pinned to `1.1.0` rather than `latest`. Login, cluster layout
initialization, bucket and key creation, permission assignment, and object
upload/list/download/delete were verified against Garage `v2.5.0` in an
isolated single-node stack. Repeat a small upload/download test after
deployment to verify your actual DNS, HTTPS routing, and storage.
