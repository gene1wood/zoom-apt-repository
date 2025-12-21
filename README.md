
# Zoom APT Repository (via GitHub Pages)

This repository automatically mirrors the latest Zoom AMD64 `.deb` package
and serves it as a static APT repository via GitHub Pages.

## Features

- Nightly GitHub Action fetches Zoom deb package
- Skips download if the package version has not changed
- Static APT repo built with `apt-ftparchive`
- Served via GitHub Pages

## Usage

* [Fork this repo in GitHub](https://github.com/gene1wood/zoom-apt-repo/fork)
* Enable GitHub pages by going to `settings/pages` and setting the 
  `Build and deployment` `Source` to `GitHub Actions`
* Generate a GPG keypair for your forked repo by running the following commands
  on your local laptop
  ```
  homedir=$(mktemp --directory)
  gpg --homedir "$homedir" --quick-generate-key --passphrase '' \
    "GitHub Actions Zoom APT Repo  <github-actions-zoom-apt@example.com>" \
    ed25519 sign 0
  gpg --homedir "$homedir" --armor --export-secret-key  # Display the private key
  rm -rf "$homedir"
  ```
* In GitHub, browse to `Settings` ... `Secrets and variables` ... `Actions`
  and click `New repository secret`
* Set the secret name to `ZOOM_APT_GPG_PRIVATE_KEY` and for the Secret value,
  copy and paste the `PGP PRIVATE KEY` you produced above.
* Either wait until the nightly GitHub action which fetches the zoom deb runs,
  or manually trigger the `Fetch Zoom DEB file nightly` GitHub action if you'd
  rather not wait.
* Configure your client's apt to use the new source with the commands below.
  Note you have to switch in your GitHub username.

```bash
YOUR_GITHUB_USERNAME=octocat  # Change this to your GitHub username
sudo curl --location --output /etc/apt/keyrings/github-actions-zoom-apt.asc \
  "https://$YOUR_GITHUB_USERNAME.github.io/zoom-apt-repo/github-actions-zoom-apt.asc"
echo "deb [signed-by=/etc/apt/keyrings/github-actions-zoom-apt.asc arch=amd64] " \
  "https://$YOUR_GITHUB_USERNAME.github.io/zoom-apt-repo stable main" \
  | sudo tee /etc/apt/sources.list.d/github-actions-zoom.list
sudo apt update
sudo apt install zoom
```

## FAQ

* Why do I need to fork the repo?
  * The reason you need your own fork (instead of just using this upstream
    gene1wood/zoom-apt-repo repo) is because GitHub bills for large file
    storage (the Zoom deb file) and bandwidth to download it (when you do an
    apt upgrade). As a result, if everyone used the upstream gene1wood/zoom-apt-repo
    then it would stop working as the billing exceeded the free tier. You can
    see the billing details including LFS storage and bandwidth for your fork
    by going to https://github.com/settings/billing
* Is this secure and safe to use?
  * By forking the code into your own repo and creating your own GPG signing
    key, you remove the ability for the upstream gene1wood/zoom-apt-repo to
    potentially inject malware into your zoom package.
* Why not use an absolute URL in the apt repository that points to the file
  hosted on the Zoom CDN?
  * It appears that it's not possible to get apt to support absolute URLs.
    I've tried with the `Filename:` field and the apparently non-existent
    `URI:` field. Using `Filename:` such that the resulting final URL which
    the apt client fetches returns a HTTP 301 redirect to the Zoom CDN would
    work, but GitHub Pages doesn't support redirects.
* Why not use a GitHub repo served up with CloudFlare Pages which does support
  redirects?
  * This would possibly raise the barrier to someone setting up their own
    instance of this (requiring setting up CloudFlare Pages as well) too far.
    I'm assuming folks will want to spin up their own instead of trusting
    the security of packages signed by someone they don't know/trust.
* Why not use ETags and look for HTTP 304 responses to determine if there's a
  new zoom package?
  * It appears that [Zoom's CDN isn't configured correctly](https://gist.github.com/gene1wood/5249118ad566eb349f4b85da2f8c9e8f) and when a valid
    ETag is sent with a request, the Zoom CDN responds with a HTTP 200 along
    with the file and even shows the same ETag in the header response.