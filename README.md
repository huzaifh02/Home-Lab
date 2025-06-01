# Homelab Kubernetes Setup

This repository contains Kubernetes manifests and documentation for my homelab setup, including Jellyfin media server with Cloudflare Tunnel integration.

## Structure

```
.
├── k3s/
│   ├── jellyfin/          # Jellyfin media server setup
│   │   ├── README.md      # Detailed setup instructions
│   │   └── service.yaml   # Jellyfin service configuration
│   └── cloudflare/        # Cloudflare Tunnel setup
│       ├── configmap.template.yaml
│       ├── deployment.yaml
│       └── namespace.yaml
└── README.md              # This file
```

## Setup Instructions

Detailed setup instructions can be found in the respective directories:

- [Jellyfin Setup with Cloudflare Tunnel](k3s/jellyfin/README.md)

## Security Notes

- This repository contains template files only
- No credentials or sensitive information are stored in the repository
- All sensitive files are listed in `.gitignore`
- When setting up your own instance, make sure to:
  1. Never commit credentials or secrets
  2. Use environment variables or secure secret management
  3. Keep your Cloudflare tunnel credentials secure

## License

MIT License - feel free to use this setup for your own homelab! 