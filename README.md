# Deploying Starcharts to Vercel

This guide documents the full deployment flow for this fork of
[`Nuyoahwjl/my-starchart`](https://github.com/Nuyoahwjl/my-starchart).

The deployed service generates star-history SVG charts for any public GitHub
repository. After deployment, the SVG endpoint can be embedded directly in a
repository `README.md`.

## How the Project Works

This project is a Go HTTP service.

Important routes:

- `/` renders the homepage.
- `/{owner}/{repo}` renders a web page for a repository.
- `/{owner}/{repo}.svg` returns the star-history SVG chart.
- `/metrics` exposes Prometheus metrics.

The SVG route is the endpoint used in README files:

```text
https://your-vercel-domain.vercel.app/owner/repo.svg
```

Example:

```text
https://your-vercel-domain.vercel.app/caarlos0/starcharts.svg?variant=adaptive
```

## Vercel Compatibility Changes

Vercel provides the runtime port through the `PORT` environment variable.
The original project reads its listen address from `LISTEN`, so `main.go`
must bridge Vercel's `PORT` into the project configuration:

```go
if port := os.Getenv("PORT"); port != "" && os.Getenv("LISTEN") == "" {
	config.Listen = ":" + port
}
```

The repository should also include a `vercel.json` file at the project root:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "framework": "go"
}
```

The GitHub API rate-limit configuration should be passed into the GitHub
client:

```go
return &GitHub{
	tokens:          roundrobin.New(config.GitHubTokens),
	pageSize:        config.GitHubPageSize,
	maxSamplePages:  config.GitHubMaxSamplePages,
	cache:           cache,
	maxRateUsagePct: config.GitHubMaxRateUsagePct,
}
```

## Required Environment Variables

Configure these variables in Vercel under:

```text
Project Settings -> Environment Variables
```

Recommended variables:

```env
REDIS_URL=rediss://default:PASSWORD@ENDPOINT:PORT
GITHUB_TOKENS=github_pat_xxx
GITHUB_PAGE_SIZE=100
GITHUB_MAX_SAMPLE_PAGES=15
GITHUB_MAX_RATE_LIMIT_USAGE=80
```

`LISTEN` should normally be left unset on Vercel. Vercel will provide `PORT`,
and the application will listen on that port automatically.

## Connecting External Redis

Redis is strongly recommended for production. Without Redis, the service may
still generate charts, but it will repeatedly call the GitHub API and is much
more likely to hit rate limits.

The easiest hosted Redis option is Upstash Redis.

Create a Redis database in Upstash or through the Vercel Marketplace, then copy
the TCP Redis connection string. For this Go project, use `REDIS_URL` with a
`rediss://` URL:

```env
REDIS_URL=rediss://default:PASSWORD@ENDPOINT:PORT
```

Example:

```env
REDIS_URL=rediss://default:AbCdEf123456@fine-mantis-12345.upstash.io:6379
```

Do not use only the REST variables for this project:

```env
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
KV_REST_API_URL=...
KV_REST_API_TOKEN=...
```

Those variables are for HTTP-based Redis clients. This project uses the Go
Redis TCP client and expects `REDIS_URL`.

## GitHub Token Setup

Create a GitHub token and set it as `GITHUB_TOKENS`.

For public repositories, a read-only token is enough. Multiple tokens can be
provided as a comma-separated list:

```env
GITHUB_TOKENS=github_pat_xxx,github_pat_yyy
```

Multiple tokens help distribute API usage when the service receives many chart
requests.

## Deploy to Vercel

1. Push the repository to GitHub:

   ```text
   https://github.com/Nuyoahwjl/my-starchart
   ```

2. Open Vercel and create a new project.

3. Import the GitHub repository.

4. Select the Go framework preset if Vercel does not detect it automatically.

5. Add the environment variables listed above.

6. Deploy the project.

After deployment, Vercel will provide a public domain such as:

```text
https://your-vercel-domain.vercel.app
```

## Generate SVG Charts

The public SVG URL format is:

```text
https://your-vercel-domain.vercel.app/{owner}/{repo}.svg
```

Examples:

```text
https://your-vercel-domain.vercel.app/caarlos0/starcharts.svg
https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart.svg?variant=adaptive
```

Supported style variants:

```text
variant=adaptive
variant=light
variant=dark
```

The recommended README variant is `adaptive`, because it works better with
GitHub light and dark themes:

```text
https://your-vercel-domain.vercel.app/owner/repo.svg?variant=adaptive
```

## Embed in a README

Use Markdown image syntax:

```md
[![Stargazers over time](https://your-vercel-domain.vercel.app/owner/repo.svg?variant=adaptive)](https://github.com/owner/repo)
```

Example:

```md
[![Stargazers over time](https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart.svg?variant=adaptive)](https://github.com/Nuyoahwjl/my-starchart)
```

## Custom Colors

The SVG endpoint supports custom colors:

- `background`
- `axis`
- `line`

Because `#` has special meaning in URLs, encode it as `%23`.

Example:

```md
[![Stargazers over time](https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart.svg?background=%23ffffff&axis=%23333333&line=%236b63ff)](https://github.com/Nuyoahwjl/my-starchart)
```

## Testing the Deployment

Open a repository page:

```text
https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart
```

Test the SVG endpoint:

```text
https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart.svg?variant=adaptive
```

If opening the SVG directly in a browser does not behave as expected, test it as
an image embed or use an explicit SVG accept header:

```bash
curl -H "Accept: image/svg+xml" \
  "https://your-vercel-domain.vercel.app/Nuyoahwjl/my-starchart.svg?variant=adaptive"
```

## Troubleshooting

### Redis connection errors

Check that `REDIS_URL` uses `rediss://` for Upstash and that the password,
endpoint, and port are correct.

### GitHub rate-limit errors

Set `GITHUB_TOKENS` and keep Redis enabled. For higher traffic, provide multiple
tokens separated by commas.

### Vercel port errors

Make sure the `PORT` compatibility block exists in `main.go` and that `LISTEN`
is not overriding it in Vercel.

### Color parameters do not work

Encode `#` as `%23` in URLs. For example, use `%236b63ff` instead of `#6b63ff`.

