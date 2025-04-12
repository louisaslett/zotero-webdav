# Zotero WebDAV

A docker container setup to run a personal WedDAV store for Zotero papers.
My use case is to host this on a personal machine which is not directly accessible from the internet, and instead to connect via a Tailscale VPN to improve security and mean regular patching/updating is less critical.

The approach here borrows heavily from <https://github.com/BytemarkHosting/docker-webdav>, making adjustments to enable signed SSL certificates via DNS-01 challenge authentication with LetsEncrypt.
This is necessary because Zotero apparently requires SSL from iOS and we want to avoid self signed cert problems.
However, the need to keep it inaccessible except via the Tailnet means we must use DNS-01 challenge.
