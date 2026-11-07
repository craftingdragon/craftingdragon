# Self-Hosted GitHub Readme Stats and Streak Stats

This builds a Docker image for `github-readme-stats` without Vercel. Secrets are provided at runtime, not baked into the image. The default base image is Node LTS 24 on Alpine 3.23. Production npm dependencies are refreshed within upstream’s declared version ranges during builds; npm, Git metadata, and development manifests are excluded from the runtime.

The Compose stack also builds [GitHub Readme Streak Stats](https://github.com/DenverCoder1/github-readme-streak-stats#-deploying-it-on-your-own) using a local PHP-FPM Alpine image (SVG cards only, without Apache or Inkscape) and runs nginx as the shared entry point.

## Build

```bash
docker build \
  -t registry.gamepowerx.com/github-readme-stats:latest \
  ./docker/github-readme-stats
```

To pin upstream to a known commit or tag:

```bash
docker build \
  --build-arg GITHUB_README_STATS_REF=<commit-or-tag> \
  -t registry.gamepowerx.com/github-readme-stats:<tag> \
  ./docker/github-readme-stats
```

## Push

```bash
docker push registry.gamepowerx.com/github-readme-stats:latest
```

## Run

Create a `.env` file next to `compose.yml` on your server. Both apps use this token (`PAT_1` for stats, `TOKEN` for streaks):

```dotenv
PAT_1=github_pat_or_classic_token_here
```

Then start it:

```bash
docker compose --env-file docker/github-readme-stats/.env -f docker/github-readme-stats/compose.yml up -d --build
```

The first build requires GitHub and container registry access. Streak source is pinned by `STREAK_REF` in its Dockerfile; change that build argument to update it. nginx forwards `/streak/` over FastCGI to the fixed PHP entry point. Other `/streak/*` paths return 404; `type=png` returns 400. The upstream demo UI is not deployed. `/streak` redirects to `/streak/` while retaining query parameters. All other paths retain their original Node routing.

All three images run with non-root UIDs, read-only root filesystems, no capabilities, and privilege escalation disabled. nginx listens internally on 8080; only its host port 9000 is published. PHP-FPM listens internally on 9000 and keeps its cache in the `streak-cache` named volume. `/tmp` is writable tmpfs. No root initialization container is required.

## Refresh and scan before deployment

Every Alpine build stage runs `apk upgrade --no-cache`. Refresh image bases and rerun package upgrades rather than reusing cached layers:

```bash
# Export PAT_1 first, or supply --env-file as in the startup command.
docker compose -f docker/github-readme-stats/compose.yml build --pull --no-cache
for image in github-readme-stats github-readme-streak-stats github-readme-nginx; do
  docker scout cves "local://registry.gamepowerx.com/$image:latest"
  docker scout cves --only-severity critical,high --exit-code "local://registry.gamepowerx.com/$image:latest" || exit 1
done
```

Critical/High findings block release, including findings without fixes. Alpine upgrades cannot patch software bundled outside apk (such as Node or PHP); update affected base images or application dependencies as needed. Scan failures are not passing results. See `security-validation.md` for the last local validation results. The push command above publishes only Node; publish the streak and nginx images under their Compose image tags as well when deploying prebuilt images, then start with `up -d --no-build`.

## README URLs

Point your existing HTTPS reverse proxy for `ghreadme.craftingdragon.ch` to nginx on host port `9000`, forwarding all paths, then use URLs like these (nginx in this stack serves HTTP):

```txt
https://ghreadme.craftingdragon.ch/api?username=craftingdragon&show_icons=true&theme=tokyonight&hide_border=true
https://ghreadme.craftingdragon.ch/api/top-langs/?username=craftingdragon&layout=compact&theme=tokyonight&hide_border=true
https://ghreadme.craftingdragon.ch/streak/?user=craftingdragon&theme=blue-green&hide_border=true
```

Keep `WHITELIST=craftingdragon` unless you intentionally want other GitHub users to use your instance.
