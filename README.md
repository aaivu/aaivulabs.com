# Aaivu Labs

Static website for [aaivulabs.com](https://aaivulabs.com) — an AI research and development company based in Colombo, Sri Lanka.

## Tech Stack

- HTML5 + Tailwind CSS (CDN)
- Font Awesome icons
- Google Fonts (Plus Jakarta Sans)
- Zero build steps — pure static files

## Local Development

```bash
cd website
python3 -m http.server 8080
# Open http://localhost:8080
```

## Deployment

Hosted on AWS S3 + CloudFront with Route 53 DNS. See [DEPLOY.md](DEPLOY.md) for the full setup guide.

Quick deploy after initial setup:

```bash
aws s3 sync website/ s3://aaivulabs.com/ --delete
aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"
```
