# Deployment validation — 2026-11-07

Final local linux/amd64 images were built and tested; no images were pushed or deployed to a server. Scan results are a point-in-time check, not a guarantee about future findings.

| Image | Docker Scout findings | Trivy findings | Runtime UID:GID |
| --- | --- | --- | --- |
| github-readme-stats | 0 | 0 | 10001:10001 |
| github-readme-streak-stats | 0 | 0 | 82:82 |
| github-readme-nginx | 0 | 0 | 10002:10002 |

All severity counts are zero. Docker Scout v1.29.0 scanned the final local images, with separate Critical/High exit-code gates returning zero. No base-image or unfixed-vulnerability exclusions were used. Trivy scanned `docker image save` tar exports using `docker.io/aquasec/trivy:latest`, with vulnerability scanning enabled for OS and application packages. The scanner received archive files and a database cache, without a Docker socket mount. No findings were suppressed.

## Exact images

- `registry.gamepowerx.com/github-readme-stats:latest`: `sha256:30286108c6f528369bcc3a9d9e6dd260bbaef417a00996ea0ec006c991e822aa`
- `registry.gamepowerx.com/github-readme-streak-stats:latest`: `sha256:0b315778696289388580e413d202df910ed3018d4de36f2cef7b33854a027048`
- `registry.gamepowerx.com/github-readme-nginx:latest`: `sha256:9f35cdf902c37c2fe46ad0e589b90af9dad8c423990b15baa31436248afd6da6`
- `aquasec/trivy:latest`: `sha256:52389f537db07a4dc0d289f61893ec791a3244d2f5cf21c819a2e874d72f3764`

## Runtime checks

- All three services started with explicit non-root users, read-only root filesystems, dropped capabilities, and `no-new-privileges:true`.
- Each PID 1 had non-root UIDs, `CapEff: 0000000000000000`, and `NoNewPrivs: 1`.
- nginx and PHP-FPM configuration checks passed.
- Both Node API routes and the streak endpoint returned SVG through nginx. Query options and the `/streak` redirect were exercised.
- A successful themed streak card was generated from a cached fixture: valid SVG XML and `Content-Type: image/svg+xml`.
- Cache writes succeeded and persisted after PHP container recreation. PNG requests returned 400; demo paths returned 404.
- Apache and Inkscape were absent from the PHP image. Compose validation and `git diff --check` passed.
- Temporary test stacks were removed. No real GitHub token was available; live GitHub contribution retrieval and public HTTPS/GitHub Camo rendering were not tested.

## Fixes prompted by scans

The official nginx image retained vulnerable nginx packages after upgrade; the proxy now installs patched nginx directly from Alpine 3.23. Node dependencies were refreshed within upstream's declared ranges, build tools and development metadata were excluded, and its base was fixed to Alpine 3.23. PHP also uses Alpine 3.23 and removes GNU tar from its runtime. Every maintained Alpine build stage includes `RUN apk upgrade --no-cache`.

## Reproduce Trivy checks

Export each final image and scan its archive (repeat for the three tags in Compose):

```bash
mkdir -p /tmp/readme-scan
image=registry.gamepowerx.com/github-readme-streak-stats:latest
docker image save -o /tmp/readme-scan/image.tar "$image"
docker run --rm -v /tmp/readme-scan:/images:ro \
  docker.io/aquasec/trivy:latest image --scanners vuln --input /images/image.tar
```

Use `--severity CRITICAL,HIGH --exit-code 1` for a separate Trivy release gate; keep the unrestricted scan to report all findings. The reproduction command uses the scanner's default user; deployed applications always run as the non-root users listed above.
