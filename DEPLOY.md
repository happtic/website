# Deploying happtic.com

The site is a single self-contained static page — `index.html` is the whole thing
(inline CSS/JS; the only external dependency is the Google Fonts stylesheet for
IBM Plex Mono). No build step.

## Hosting architecture (AWS)

```
Route53 (happtic.com)  ->  CloudFront (HTTPS + redirects)  ->  S3 (private, OAC)
```

- **S3** bucket `happtic-com-site` in `ap-southeast-2`, **private** — reachable only
  through CloudFront via an Origin Access Control (OAC). Public access is blocked.
- **CloudFront** distribution (`d8dmazmy3d4vj.cloudfront.net`) serves the bucket over
  HTTPS with:
  - **HTTP → HTTPS** redirect (viewer protocol policy `redirect-to-https`).
  - **www → apex** 301 redirect via a CloudFront Function (`happtic-www-to-apex`);
    the site is **no-www canonical** (`https://happtic.com`).
  - Default root object `index.html`.
- **ACM** certificate in **us-east-1** (required region for CloudFront) covering
  `happtic.com` and `www.happtic.com`, DNS-validated via Route53.
- **Route53** zone `happtic.com`: apex and `www` `A`/`AAAA` **alias** records point at
  the CloudFront distribution. (Mail records — Proton MX/SPF/DKIM/DMARC — are separate
  and must not be touched by deploys.)

## Publishing a change

```bash
# 1. upload the page
aws s3 cp index.html s3://happtic-com-site/index.html \
  --content-type "text/html; charset=utf-8"

# 2. invalidate the CloudFront cache so the edit shows immediately
DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Aliases.Items[0]=='happtic.com'].Id" --output text)
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/index.html" "/"
```

Run these from a shell authenticated to the AWS account that owns the resources above
(region `ap-southeast-2`).

## Notes

- If the apex ever appears not to resolve right after a DNS change, it's usually a
  cached negative (`NXDOMAIN`) answer on your local resolver — check a public resolver
  with `dig +short @1.1.1.1 happtic.com` and flush your local DNS cache.
- Account-specific identifiers (AWS account id, cert ARN, IAM principals) are
  intentionally omitted from this public file; look them up in the AWS console/CLI.
