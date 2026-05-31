# Runbook

How to do stuff.

## Prerequisites

- **Java 25** (JColor `v6.*` targets Java 25). Install via `brew install openjdk@25` and add to your shell PATH/`JAVA_HOME`.
- **Maven 3.9+** (`brew install maven`).
- **`~/.m2/settings.xml`** with a `<server id="central">` block holding your Sonatype Central Portal user token. Generate the token at https://central.sonatype.com/account.
- **GPG secret key** present in `gpg --list-secret-keys`, with its fingerprint matching `<keyname>` in `pom.xml`. Public half published to `keyserver.ubuntu.com` and `keys.openpgp.org`.

## Release a new version

1. Make sure `main` is green locally:
   - `mvn clean test`
2. Bump the version and tag:
   - `mvn release:clean release:prepare`
   - When prompted for the release version, follow semver:
     - Breaking changes → increase major (`X.*.*`)
     - New features or dependency upgrades → minor (`*.Y.*`)
     - Bug fixes → patch (`*.*.Z`)
3. Build, sign, upload, and publish to Sonatype Central Portal:
   - `mvn release:perform`
   - Enter your GPG passphrase when prompted.
   - `autoPublish=true` in `pom.xml` makes the deployment auto-promote
     once validation passes — no manual click required.
   - The artifact lands on `repo.maven.apache.org` ~10–30 minutes later.
   - You can watch progress at https://central.sonatype.com/publishing/deployments.
4. Push tags and the bump commits:
   - `git push --follow-tags`
5. Regenerate and commit the published Javadoc (see [Generate Javadoc](#generate-javadoc)).
6. On GitHub, draft a release at https://github.com/dialex/JColor/releases:
   - Select the tag created by `release:prepare`.
   - Describe the changes (highlight breaking changes if any).

## Generate Javadoc

The site at https://dialex.github.io/JColor/ is served from the `docs/` folder on `main`. Refresh it after each release.

```sh
./.github/update-javadoc.sh
git add docs
git commit -m "doc: update javadoc to vX.Y.Z"
git push
```

The script runs `mvn javadoc:javadoc`, copies `target/reports/apidocs/` into `docs/`, and opens the result in a browser for visual sanity-check.

## Update dependencies

- Check what is outdated: `mvn versions:display-dependency-updates versions:display-plugin-updates`
- Bump individual entries in `pom.xml` (avoid `versions:use-latest-releases` — it can pick incompatible pre-releases).
- Re-run the updates command — some plugin upgrades unlock further upgrades that were gated by the previous Maven floor.
- Confirm tests still pass: `mvn clean test`.

## Generate a new GPG signing key

Only needed if the existing key was lost, expired, or compromised.

1. Create the key:
   - `gpg --full-generate-key`
   - Pick RSA + RSA, 4096 bits, 2-year expiry (or no expiry), real name, an email tied to your public identity.
2. Find the new fingerprint:
   - `gpg --list-secret-keys --keyid-format=long` — copy the 40-char hex under `sec`.
3. Publish the public half:
   - `gpg --keyserver keyserver.ubuntu.com --send-keys <FINGERPRINT>`
   - `gpg --keyserver keys.openpgp.org --send-keys <FINGERPRINT>`
4. Update `<keyname>` in `pom.xml` with the new fingerprint.
5. Back up the secret key, ownertrust, and a revocation cert:
   - `gpg --armor --export-secret-keys <FINGERPRINT> > jcolor-secret.asc`
   - `gpg --export-ownertrust > jcolor-trust.txt`
   - `gpg --gen-revoke <FINGERPRINT> > jcolor-revocation.asc`
   - Store all three (plus the passphrase) in a password manager. Delete the local files afterwards.
6. Smoke-test signing locally before a release:
   - `mvn clean verify` — confirm `target/*.asc` files are produced for the jar, sources jar, javadoc jar, and pom.
