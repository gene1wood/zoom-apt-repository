# Zoom APT Repository (via GitHub Pages)

[![Fetch Zoom DEB package and create apt repo](https://github.com/gene1wood/zoom-apt-repository/actions/workflows/create-apt-repo.yml/badge.svg)](https://github.com/gene1wood/zoom-apt-repository/actions/workflows/create-apt-repo.yml)

## Why is this repo archived?

This repo, which uses GitHub Actions and GitHub pages to maintain a remote
APT repo in GitHub with the most recent Zoom `.deb` pacakge, works, however
there is an easier way to accomplish this without relying on any remote APT
repo.

Instead, you can use the [`APT::Update::Pre-Invoke`](https://manpages.debian.org/stretch/apt/apt.conf.5.en.html#HOW_APT_CALLS_DPKG(1))
feature in `apt.conf` to instruct `apt` to check for an updated Zoom `.deb`
file and fetch it if needed. This solution is simpler as it only requires a
local file and removes GitHub Actions and GitHub Pages from the picture.

To enable this simpler approach, copy paste the code below into a file and run
it once. The code will

* Create a new `/usr/local/zoom-deb-files` directory to contain the Zoom `.deb`
  file and the APT repo files (`Packages` and `Release`)
* Create the new `/etc/apt/apt.conf.d/99update_zoom` file. This file instructs
  `apt` to check for a newer file hosted on Zoom's CDN, each time you run 
  `apt update`, and if one is present, download it and rebuild the local APT
  repo.
* Create the `/etc/apt/sources.list.d/local-zoom.list` file to instruct `apt`
  to use this new local APT repo

```shell
#!/usr/bin/env bash

# Based on https://askubuntu.com/a/1316231/14601

url=https://zoom.us/client/latest/zoom_amd64.deb
debdir=/usr/local/zoom-deb-files
aptconf=/etc/apt/apt.conf.d/99update_zoom
sourcelist=/etc/apt/sources.list.d/local-zoom.list

sudo mkdir -p $debdir
# --timestamping only fetches the file if the last-modified HTTP header in the
#   response is newer than the last modified time of the local file
# --no-if-modified-since tells wget to send a HEAD request to determine if
#   there's a new file or not before sending a GET request
# The grep prevents running apt-ftparchive if no new file was downloaded
echo 'APT::Update::Pre-Invoke ' \
'{"cd '$debdir' ' \
'&& wget --timestamping --no-if-modified-since '$url' 2>&1 ' \
'| ( grep --quiet ' "'Server file no newer than local file'" ' && exit 1 || exit 0 ) ' \
'&& apt-ftparchive packages . > Packages ' \
'&& apt-ftparchive release . > Release ' \
'|| true";};' | sudo tee $aptconf
echo 'deb [trusted=yes lang=none] file:'$debdir' ./' | sudo tee $sourcelist
```

Once this is done, you can run this to install Zoom

```shell
sudo apt update
sudo apt install zoom
```

# Deprecated Instructions

Below is the original README that gives details about the now deprecated
GitHub Actions and GitHub Pages method.

## What is this repo?

Zoom doesn't provide an official APT repository, which means keeping Zoom
updated on Debian/Ubuntu systems requires manually downloading and installing
new `.deb` packages. This project automatically mirrors the latest Zoom package
and serves it as a proper APT repository, enabling automatic updates through
your standard package manager.

This repository is designed to be forked. Each user maintains
their own copy to avoid bandwidth and storage costs, and to maintain security
by controlling their own GPG signing keys.

## Features

- Nightly GitHub Action automatically fetches the latest Zoom AMD64 `.deb`
  package
- Intelligent version checking - only downloads when Zoom releases an update
- Static APT repository built with `apt-ftparchive`
- Served free via GitHub Pages
- GPG-signed packages for security

## Architecture Support

AMD64 (x86_64) only. Zoom only provides a 64-bit x86 package.
ARM and 32-bit systems are not supported.

## Prerequisites

- A GitHub account
- A Debian/Ubuntu-based Linux system (for installation)
- GPG installed on your local machine (for one-time key generation)

## Initial Setup

### 1. Fork this Repository

[Click here to fork this upstream repository](https://github.com/gene1wood/zoom-apt-repository/fork)
to your own GitHub account.

### 2. Enable GitHub Pages

1. Go to your forked repository
2. Navigate to `Settings` ... `Pages`
3. Under `Build and deployment`, set `Source` to `GitHub Actions`

### 3. Generate a GPG Signing Key

Run these commands on your local machine to create a GPG keypair that will sign
your packages:

```bash
# Create a temporary directory for GPG operations
homedir=$(mktemp --directory)

# Generate a new GPG key (no passphrase for automation)
gpg --homedir "$homedir" --quick-generate-key --passphrase '' \
  "GitHub Actions Zoom APT Repo <github-actions-zoom-apt@example.com>" \ 
   ed25519 sign 0

# Display the private key (you'll need this in the next step)
gpg --homedir "$homedir" --armor --export-secret-key

# Clean up the temporary directory
rm -rf "$homedir"
```

Copy the entire output into your clipboard, including the
`-----BEGIN PGP PRIVATE KEY BLOCK-----` and
`-----END PGP PRIVATE KEY BLOCK-----` lines.

### 4. Add the GPG Key to GitHub Secrets

1. In your forked repository, go to `Settings` ... `Secrets and variables` ...
   `Actions`
2. Click `New repository secret`
3. Set the secret name to: `ZOOM_APT_GPG_PRIVATE_KEY`
4. Paste the private key output from step 3 above as the value
5. Click `Add secret`

### 5. Trigger the Initial Build

You have two options:

Option A - Wait: The GitHub Action runs automatically every night.

Option B - Manual trigger:

1. Go to the `Actions` tab in your repository
2. Select `Fetch Zoom DEB package and create apt repo`
3. Click the `Run workflow` dropdown
4. Click the `Run workflow` button

The action will take a few minutes to complete. Once finished, your APT
repository will be available.

## Installing Zoom on Client Machines

After your repository is set up, configure any Debian/Ubuntu system to use it:

```bash
# Set your GitHub username
YOUR_GITHUB_USERNAME=octocat  # CHANGE THIS to your actual GitHub username

# Download and install the GPG public key
sudo curl --location --output /etc/apt/keyrings/github-actions-zoom-apt.asc \
  "https://$YOUR_GITHUB_USERNAME.github.io/zoom-apt-repository/github-actions-zoom-apt.asc"

# Add the APT repository source
echo "deb [signed-by=/etc/apt/keyrings/github-actions-zoom-apt.asc arch=amd64] " \
  "https://$YOUR_GITHUB_USERNAME.github.io/zoom-apt-repository stable main" \
  | sudo tee /etc/apt/sources.list.d/github-actions-zoom.list

# Update package lists
sudo apt update

# Install Zoom
sudo apt install zoom
```

### Verify Installation

```bash
# Check that Zoom is installed
zoom --version

# Verify updates will come from your repository
apt policy zoom
```

## Keeping Zoom Updated

Once configured, Zoom will automatically update with your regular system
updates:

```bash
sudo apt update && sudo apt upgrade
```

The GitHub Action checks for new Zoom versions nightly, so your repository will
typically have the latest version within 24 hours of Zoom releasing an update.

## Uninstalling

To remove this APT source and return to manual Zoom management:

```bash
# Remove the APT source
sudo rm /etc/apt/sources.list.d/github-actions-zoom.list sudo
rm /etc/apt/keyrings/github-actions-zoom-apt.asc

# Update package lists
sudo apt update

# Zoom will remain installed but won't receive updates from this source To
# remove Zoom entirely: sudo apt remove zoom
```

## Security Considerations

### Trust Model

When you fork this repository and generate your own GPG key, you are:

- Trusting: Zoom's official download server (where the package
  originates)
- Trusting: Your own GitHub account security and GPG key management
- Trusting: The code in your forked repository (which you can audit)
- NOT trusting: The upstream `gene1wood/zoom-apt-repository` maintainer to
  sign your packages

This setup gives you control over the entire supply chain from Zoom's servers to
your systems.

It's a good practice to audit the code in your fork, especially the GitHub
Action workflow

## FAQ

### Why do I need to fork the repo?

GitHub charges for large file storage (LFS) and bandwidth usage. The Zoom `.deb`
package is large, and if everyone used the upstream repository, costs would
quickly exceed the free tier. By maintaining your own fork, you're responsible
for your own storage and bandwidth within GitHub's free tier limits.

You can monitor your usage at https://github.com/settings/billing

### Is this secure and safe to use?

Yes, when properly configured. By forking the repository and generating your own
GPG signing key, you maintain complete control over the package signing
process. The upstream repository cannot inject malicious code into packages.
You can audit all code in your fork to verify it only downloads from Zoom's
official servers.

### Why not use absolute URLs pointing directly to Zoom's CDN?

APT doesn't support absolute URLs in repository metadata in a way that would
allow this. While you can specify alternative URLs, they must be relative to
the repository root. GitHub Pages also doesn't support HTTP redirects, which
would be another potential solution.

### Why not use CloudFlare Pages with redirects?

While CloudFlare Pages supports redirects, this would increase the setup
complexity and create another dependency. The goal is to keep this as simple as
possible - just GitHub and APT - to lower the barrier for users who want to
maintain their own secure package source.

### Why not use ETags to detect updates?

Zoom's CDN appears to be misconfigured for ETag-based caching. When sending a
valid ETag, the server returns HTTP 200 with the full file instead of HTTP 304
(Not Modified), making ETag-based update detection ineffective. See
[this analysis](https://gist.github.com/gene1wood/5249118ad566eb349f4b85da2f8c9e8f)
for more information.

### How much does this cost?

Nothing. GitHub provides free:
- Repository hosting
- GitHub Actions (2,000 minutes/month for free accounts)
- GitHub Pages hosting
- Bandwidth for public repositories

## Contributing

Contributions to improve this repository are welcome! Please open an issue or
pull request on the upstream repository at
https://github.com/gene1wood/zoom-apt-repository

## License

This project is provided under the [GPLv3 GNU General Public License](LICENSE.txt)

---

This is an unofficial tool and is not affiliated with, endorsed by, or
supported by Zoom Video Communications, Inc.