# PPUA University Website

Responsive static website prepared for the PPUA MIS midterm demonstration.

## Live demo

https://liiindaa.github.io/ppua-university-website/

## Project structure

```text
index.html                          Public website entry point
assets/                             Website images, fonts, and logo
docs/                               Midterm screenshot runbook
infrastructure/kubernetes/          K3s deployment manifest
infrastructure/nginx/               Reverse-proxy configuration
```

## Run locally

```bash
python -m http.server 4173
```

Then open `http://127.0.0.1:4173/`.
