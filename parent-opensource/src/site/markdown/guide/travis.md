<!---
# This file is part of the ChillDev-Parent.
#
# @license http://mit-license.org/ The MIT license
# @copyright 2016 © by Rafał Wrzeszcz - Wrzasq.pl.
-->

# Travis-based deployment pipeline

**Note:** This page plays role mostly of the internal guideline for **Chillout Development** projects, but it's entirely general and you can re-use the same flow with your own projects.

**Note:** **VersionEye** plugin is not part of the deployment pipeline, just a regular build plugin, but as it requires configuration through environment variable it's included here.

## Concept

### Principles

-   When possible rely on **Maven**, rather than CI to avoid lock-in with particular CI solution.
-   Try to automate as much as possible, but no more.

### Actions

Our current deployment involves following specific actions:

-   Signing artifacts with GPG key.
-   Publishing artifacts (together with GPG checksums) to Sonatype Nexus.
-   Publishing GitHub page.
-   Creating Git tag.

### Approach

The flow is as follows - when the release is about to happend, it has to be prepared on `master` branch (the contnet of the repository should reflect the desired release state), all versions must be set (preferable with `mvn versions:set` plugin call) to desired value (must not contain `-SNAPSHOT` suffix). Such prepared commit goes to `master` branch and after pushing to **GitHub** repository it triggers the build on **Travis** which involves deployment cycle.

**Note:** Commits to master should be done only when the stable release is meant to happen with the release version set in `pom.xml` file.

**Note:** We decided to rely on `master` branch instead of a tag, as tag creation can be easily automated. Additionally having tagging as part of deployment process can prevent us from creating tags when the build doesn't succeed.

As major and minor releases usually demand some manual changes it's release process need to be initialized manually. However for current active version (being stored on branch `develop`), each update automatically triggers a release version update with full deployment cycle.

## Setup

As the deployment process relies on some sensitive data, like **Sonatype** credentials, **GitHub** token etc., to pass them safely into the build we use environment variables encoded with **Travis** repository key. To pass large blobs like keys, we ue additional secret string that we randomly generate on our own for each repository, store it encoded as well and use as a key to encrypt/decrypt files with `openssl`.

**Note:** Keep in mind that the private key of the pair is never exposed, even to us, so whenever you want to encode data that may be later needed by you keep the copy of it.

To keep our project **Maven** setup independent from the choosen deployment pipeline we keep dedicated `settings.xml` for **Travis** deploys that populates the credentials and other properties.

**Note:** Everything related to **Travis**-based deployment should go into `.travis/` subdirectory.

To prepare the repository to use **Travis** for deployment automation first of all pick a secret string for it (keep it!), for example using (replace any non-alphanumerical character of the output with random character to avoid possible escaping):

```bash
cat /dev/urandom | base64 | head -c 16
```

Export it as a shell environment variable to make it reusable in further calls:

```bash
export SECRET=<secret string>
```

### Keys

Use the secret to encode key files. We need **SSH** public key for **GitHub** access and **GPG** key for signing artifacts to be deployed.

To generates **SSH** key use:

```bash
ssh-keygen -t rsa -b 4096 -C "Travis CI"
```

-   Copy public key and put it as a deployment key of your **GitHub** repository (grant it write access - it will be used for pushing tags).
-   Encrypt private key file:

```bash
openssl aes-256-cbc -k "$SECRET" -in .travis/id_rsa -out .travis/id_rsa.enc
```

To export and encrypt **GPG** (the one associated with **Sonatype** account, for tutorial about generating the key itself use **Google**) do:

```bash
gpg -a --export deployer@account.com > .travis/gpg_key
gpg -a --export-secret-keys deployer@account.com >> .travis/gpg_key
openssl aes-256-cbc -k "$SECRET" -in .travis/gpg_key -out .travis/gpg_key.enc
```

**Note:** Never, ever commit unencrypted keys to the repositories (not even private onces)!

### Environment variables

To pass the short, or variable data into deployment builds we use encrypted environment variables. Following environment variables are used:

-   `VERSIONEYE_API_KEY` - **VersionEye** API key;
-   `OSSRH_USERNAME` - **Sonatype** deployments accoung login;
-   `OSSRH_PASSWORD` - **Sonatype** deployments accoung password;
-   `GPG_PASSPHRASE` - password for **GPG** key;
-   `GITHUB_TOKEN` - **GitHub** access token, used for deploying pages;
-   `SECRET` - our secret scring used for decrypting key files.

To generate encrypted values for them call:

```bash
travis encrypt VERSIONEYE_API_KEY=<VersionEye API key>
travis encrypt OSSRH_USERNAME=<Sonatype account login>
travis encrypt OSSRH_PASSWORD=<Sonatype account password>
travis encrypt GPG_PASSPHRASE=<GPG key passphrase>
travis encrypt GITHUB_TOKEN=<GitHub access token>
travis encrypt SECRET=$SECRET
```

Save produced strings in `.travis.yml`:

```yaml
env:
    global:
        -
            secure: …
        -
            secure: …
        …
```

### Plain files

To bind the described setup with **Travis** build cycle you will need two files - there is no point in describing them here as the description may most likely go outdated and miss the real requirements, so the best way is to just look into this repository examples and use them as a templates:

-   `.travis.yml` - sections `before_deploy` to prepare required deployment environment and `deploy` for actual deployment command;
-   `.travis/settings.xml` - file that contains mapping of environment variables into **Maven** properties of the deployment configuration;
-   `.travis/release.sh` - script containing CD pipeline logic as **Travis** doesn't support complex deployment scripts.