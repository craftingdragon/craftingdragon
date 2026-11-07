# Deployment validation — 2026-11-07

Final local linux/amd64 images were built and tested. Nothing was pushed or deployed to a server. Scan results describe these exact images at the time of scanning.

| Image | Docker Scout findings | Trivy findings | Runtime UID:GID |
| --- | --- | --- | --- |
| github-readme-stats | 0 | 0 | 10001:10001 |
| github-readme-streak-stats | 0 | 0 | 82:82 |
| github-readme-nginx | 1 High: musl backport false positive | 0 | 101:101 |

## Exact images

- `registry.gamepowerx.com/github-readme-stats:latest`: `sha256:30286108c6f528369bcc3a9d9e6dd260bbaef417a00996ea0ec006c991e822aa`
- `registry.gamepowerx.com/github-readme-streak-stats:latest`: `sha256:0b315778696289388580e413d202df910ed3018d4de36f2cef7b33854a027048`
- `registry.gamepowerx.com/github-readme-nginx:latest`: `sha256:0b4007daff9b85c3f77eee3ae15e8cb6d3934e8e68eeb6d665f0ee920c6c441f`
- `aquasec/trivy:latest`: `sha256:52389f537db07a4dc0d289f61893ec791a3244d2f5cf21c819a2e874d72f3764`

## Proxy module removal

The proxy retains exactly `FROM nginxinc/nginx-unprivileged:alpine3.24`. Before upgrading packages, its Dockerfile removes `nginx-module-xslt`, `nginx-module-njs`, `nginx-module-geoip`, and `nginx-module-image-filter`. apk removes their orphaned dependencies, including libxml2/libxslt. Build assertions fail if either XML library remains installed.

A final-container check confirmed all four module packages and both XML libraries are absent. Scout indexed 38 packages, down from 88 before removal. Neither scanner reports libxml2 findings in this image. Every maintained Alpine stage still runs `RUN apk upgrade --no-cache`; nginx is explicitly upgraded to replace its base-image apk pin. All deployed processes remain non-root.

The former libxml2 finding, CVE-2026-86140, affected a DTD-validation error formatter ([upstream fix](https://github.com/GNOME/libxml2/commit/d1686f91dbda141a752200419d35639fd6b38340)). Before removal, the library was already outside the configured execution path: nginx did not link it or load XSLT/njs modules, and its proxy routes forwarded SVG without XML parsing. Removing the modules now removes the affected package entirely from the final filesystem.

## Remaining musl scanner finding

Scout v1.29.0 reports CVE-2026-107891 against musl 1.2.6-r3, comparing it to revision 1.2.6-r6. [Alpine's authoritative security tracker](https://security.alpinelinux.org/vuln/CVE-2026-107891) marks **3.24-main musl 1.2.6-r3 as fixed**; the r6 revision applies to edge-main. This is a stable-branch backport false positive, not an unpatched finding for this image. This assessment does not generalize to other musl CVEs.

The unmodified Scout Critical/High gate still returns exit code 2; no findings have been suppressed. Trivy reports zero findings at every severity. Node/PHP retain their previous zero-finding scans. Trivy used `docker.io/aquasec/trivy:latest` to scan fresh `docker image save` tar exports, with vulnerability scanning enabled for OS and application packages, without mounting the Docker socket.

## Runtime verification

The reduced nginx image was tested with the complete Compose configuration in a temporary isolated stack:

- All services ran with explicit non-root users, read-only root filesystems, dropped capabilities and `no-new-privileges:true`. PID 1 checks showed non-root UIDs, zero effective capabilities and `NoNewPrivs: 1`.
- nginx and PHP-FPM configuration checks passed. No nginx log-permission alerts appeared; startup logs use `nginx -e /dev/stderr`.
- Both Node routes and the streak endpoint returned SVG through nginx; query options and the slash redirect were exercised.
- A successful themed streak card from a cached fixture was valid SVG XML with `Content-Type: image/svg+xml`.
- Cache writes and persistence after PHP recreation passed. PNG requests returned 400 and demo routes returned 404.
- Compose validation and `git diff --check` passed; temporary test stacks were removed.

No real GitHub token was available: live contribution retrieval and public HTTPS/GitHub Camo rendering were not tested.

## Reproduce Trivy checks

Export each final image and scan its archive (repeat for all three Compose image tags):

```bash
mkdir -p /tmp/readme-scan
image=registry.gamepowerx.com/github-readme-nginx:latest
docker image save -o /tmp/readme-scan/image.tar "$image"
docker run --rm -v /tmp/readme-scan:/images:ro \
  docker.io/aquasec/trivy:latest image --scanners vuln --input /images/image.tar
```

Use `--severity CRITICAL,HIGH --exit-code 1` for a separate Trivy gate; keep an unrestricted scan to report all findings. Scanner failures are never passing results. The reproduction command uses the scanner's default user; the deployed applications always use the non-root users listed above.
