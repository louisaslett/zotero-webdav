# Zotero WebDAV

This is a Docker image to enable self-hosting a file sync server for Zotero for my personal use.
I'm making this available in case others find it useful for them and have tried to make it somewhat generic.
The image setup borrows heavily from <https://github.com/BytemarkHosting/docker-webdav>, but that image appears not to be maintained any more and has been pinned to a very old version of Alpine.
Also, `docker scout cves` shows it includes critical security vulnerabilities.

## Motivation

*TLDR; jump to [Usage](#usage)*

My reasons for creating this new version are twofold:

- when using WebDAV for Zotero, the iOS versions [seem to require an SSL connection and valid signed SSL certificate](https://forums.zotero.org/discussion/114747/the-zotero-app-on-ios-cannot-connect-to-the-synology-systems-webdav) (self-signed won't do);
- I want to host the WebDAV behind a firewall, with access over a VPN (specifically the brilliant [Tailscale](https://tailscale.com/) service) so that I don't need to worry about constantly updating for security issues in Apache

The natural solution to the first requirement is to get a free [Let's Encrypt](https://letsencrypt.org/) certificate.
However, the second requirement makes that slightly harder than usual, since the Let's Encrypt ACME-Challenge which validates your ownership of the site usually depends on their servers accessing a particular page hosted at the domain.
If we want the server to be accessible via VPN only then this is not an option, so we need to use [DNS-01 Challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge) instead.
I host my domain on [AWS Route 53](https://aws.amazon.com/route53/) which is supported by a [plugin](https://certbot-dns-route53.readthedocs.io/en/latest/) for the automated [Certbot](https://eff-certbot.readthedocs.io/en/stable/).

However, the old [BytemarkHosting/docker-webdav](https://github.com/BytemarkHosting/docker-webdav) images do not support packaged versions of the certbot plugin, which [does appear](https://pkgs.alpinelinux.org/packages?name=certbot-dns-route53&repo=&arch=&origin=&flagged=&maintainer=) in newer Alpine releases!
Alas, it is not as simple as rebuilding the image with a newer base layer because the image is [broken if building against newer versions](https://github.com/BytemarkHosting/docker-webdav/issues/8#issuecomment-480629432).

Hence, this image involved enough changes that I thought it worth sharing in case others find it useful.

## Usage

Pull the container:

```
docker pull ghcr.io/louisaslett/zotero-webdav:latest
```

These instructions assume you have the same use case as me.
That is, hosting the WebDAV server behind a firewall, accessing it over a VPN, with automatically setup SSL certificate via DNS-01 challenge authentication for a domain on AWS Route 53.

If you do not plan to use Zotero on iOS devices, you might be able to get away without the SSL bit, in which case see the [non-SSL setup](#non-ssl-setup).

Launch as follows, replacing all placeholders denoted by `<these_markers>`. 

```
docker run --rm \
  --name zotero \
  -p <tailscale_ip>:<tailscale_https_port>:443 \
  -e AUTH_TYPE=Digest \
  -e USERNAME=<username> \
  -e PASSWORD=<password> \
  -e SSL_CERT=certbot-dns-route53 \
  -e AWS_ACCESS_KEY_ID=<your_route53_aws_access_key> \
  -e AWS_SECRET_ACCESS_KEY=<your_route53_aws_secret_key> \
  -e SSL_DOMAIN=<your_route53_hosted_domain> \
  -e SSL_EMAIL=<your_email> \
  -v <path_on_host_for_zotero>:/var/lib/dav \
  ghcr.io/louisaslett/zotero-webdav
```

Most of the markers to replace should be self explanatory.
Note that the Tailscale IP will be the VPN IP address of the machine running the container, which will be in the `100.x.y.z` range.
You should be safe to add this address as an `A` record in your AWS Route 53 setup, since [Tailscale IP addresses are stable](https://tailscale.com/kb/1033/ip-and-dns-addresses) for your account.
The `<tailscale_https_port>` is the port you want to access it on, so can be anything that does not clash with other services running on that Tailnet node.

If you want to run the container in the background, add `-d` after `docker run`.
Personally I tend to run the container in a `screen` session so that I can easily check in on it, because there is [strange Docker attach/detach behaviour on Mac](https://github.com/docker/for-mac/issues/1598).

### AWS Keys

The AWS keys you create should be for an IAM user profile with the following IAM permissions (again, replacing `<your_hosted_zone_id>` with your zone id):

```
{
    "Version": "2012-10-17",
    "Id": "certbot-dns-route53 policy",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect" : "Allow",
            "Action" : [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource" : [
                "arn:aws:route53:::hostedzone/<your_hosted_zone_id>"
            ]
        }
    ]
}
```

### Non-SSL setup

As long as you're running over a Tailscale VPN, this should be perfectly fine and just as secure as the full blown SSL setup above.
The caveat is that Apple iOS devices probably won't work with it, as they enforce SSL.

```
docker run --rm \
  --name zotero \
  -p <tailscale_ip>:<tailscale_http_port>:80 \
  -e AUTH_TYPE=Digest \
  -e USERNAME=<username> \
  -e PASSWORD=<password> \
  -v <path_on_host_for_zotero>:/var/lib/dav \
  ghcr.io/louisaslett/zotero-webdav
```

# Troubleshooting

The setup presented here is what I am using myself, though I would be delighted if it helps someone else too.

Should you encounter issues, the most likely problem will be the SSL certificate setup.
The first time you do this, I recommend having the AWS Route 53 console open on your hosted zone and refreshing the zones panel as you launch the container, looking for the `TXT` record that Certbot should create to authenticate your ownership of the domain.
If you get stuck, the best thing is probably to attach to the container and look at the Let's Encrypt log file.

Attach using:

```
docker exec -it zotero /bin/bash
```

The logs will be stored at `/var/log/letsencrypt`.
