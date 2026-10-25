# Self-Hosted GitHub Readme Stats

This builds a Docker image for `github-readme-stats` without Vercel. Secrets are provided at runtime, not baked into the image. The default base image is Node LTS 24 Alpine.

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

Create a `.env` file next to `compose.yml` on your server:

```dotenv
PAT_1=github_pat_or_classic_token_here
```

Then start it:

```bash
docker compose -f docker/github-readme-stats/compose.yml up -d
```

## README URLs

Put the service behind HTTPS with your reverse proxy, then use URLs like:

```txt
https://stats.example.com/api?username=craftingdragon&show_icons=true&theme=tokyonight&hide_border=true
https://stats.example.com/api/top-langs/?username=craftingdragon&layout=compact&theme=tokyonight&hide_border=true
```

Keep `WHITELIST=craftingdragon` unless you intentionally want other GitHub users to use your instance.
