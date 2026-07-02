---
title: "Publishing an Android SDK to Maven Central in 2026"
date: 2026-04-29
tags: ["android", "sdk", "maven-central", "kotlin", "gradle", "open-source"]
categories: ["Android", "SDK"]
draft: false
description: "A real-world walkthrough of publishing a multi-module Android SDK to Maven Central in 2026 — including everything that broke, confused me and eventually worked."
---

This post is a real-world walkthrough of how I published my Translate Android SDK to Maven Central in 2026 — including everything that broke, confused me and eventually worked.

The repo is also open for contributions — feel free to explore, break things and improve it!!

## What we're building

We're going to build and publish a multi-module SDK:

- `translate` → Core translation SDK
- `translate-ui` → Jetpack Compose UI layer

Published as:
io.github.kanake10:translate:1.1.0

io.github.kanake10:translate-ui:1.1.0

## Step 1 — Create a Maven Central account

Go to:

https://central.sonatype.com

Sign in using GitHub or email. (I used GitHub)

This gives you:

- Publishing dashboard
- Deployment tracking
- Namespace management

## Step 2 — Create and verify your namespace

Maven Central requires a namespace (your group ID).

- Add namespace in Maven Central
- Receive verification token
- Add TXT record to DNS (or GitHub ownership verification)
- Click verify

Once verified, you own the namespace.

Example:
io.github.yourusername

This becomes your dependency prefix:
for example mine is `io.github.kanake10`

## Step 3 — Generate GPG keys (this is where I struggled the most)

Maven Central requires all artifacts to be signed.

**Install GPG (Mac)**

```bash
brew install gnupg

gpg --full-generate-key
```

Select:

- RSA and RSA
- 4096-bit key
- No expiration (or your preference)

Afterwards, you'll need to add the following. Note down your Passphrase

- Real Name
- Email
- Passphrase

**Export key for Gradle**

```bash
gpg --keyring secring.gpg --export-secret-keys > ~/.gnupg/secring.gpg
```

**List your keys**

```bash
gpg --list-secret-keys --keyid-format=long
```

## Step 4 — Local Gradle configuration (IMPORTANT)

Create this file (on your local machine):
~/.gradle/gradle.properties

**Add credentials**

```properties
mavenCentralUsername=YOUR_USERNAME
mavenCentralPassword=YOUR_PASSWORD
signing.keyId=YOUR_KEY_ID
signing.password=YOUR_GPG_PASSPHRASE
signing.secretKeyRingFile=/Users/yourname/.gnupg/secring.gpg
```

## Step 5 — Add Maven publishing plugin

We use:

```kotlin
id("com.vanniktech.maven.publish") version "0.36.0"
```

Before you even add this plugin, make sure to read about it here. It will save you from a lot of pain.

**Why this plugin exists**

Without it, publishing requires:

- manual POM generation
- manual signing setup
- manual upload flow

This plugin automates all of that and reduces mistakes.

## Step 6 — Configure module publishing

Example in my `translate` build.gradle.kts file

```kotlin
mavenPublishing {
    publishToMavenCentral()

    signAllPublications()

    coordinates(
        groupId = "io.github.kanake10",
        artifactId = "translate",
        version = "1.1.0"
    )

    pom {
        name.set("Translate SDK")
        description.set("Core translation SDK")
        url.set("https://github.com/kanake10/translate-android")

        licenses {
            license {
                name.set("Apache-2.0")
                url.set("https://www.apache.org/licenses/LICENSE-2.0.txt")
            }
        }

        developers {
            developer {
                id.set("kanake10")
                name.set("Ezra Kanake")
                url.set("https://github.com/kanake10")
            }
        }

        scm {
            url.set("https://github.com/kanake10/translate-android")
            connection.set("scm:git:git://github.com/kanake10/translate-android.git")
            developerConnection.set("scm:git:ssh://git@github.com/kanake10/translate-android.git")
        }
    }
}
```

## Step 7 — Get Maven Central tokens

From:

https://central.sonatype.com/usertoken

Generate:

- Username token
- Password token

Don't forget to add them to your `gradle.properties` file

## Step 8 — Publish

Run:

```bash
./gradlew publishToMavenCentral
```

If everything is correct:

- artifacts are signed
- uploaded
- validated
- published

## Common issues I hit

**❌ Missing username**

→ wrong property names in `gradle.properties`.
The vanniktech plugin also requires your username and password in `gradle.properties` to be exactly:

```properties
mavenCentralUsername=YOUR_USERNAME
mavenCentralPassword=YOUR_PASSWORD
```

**❌ Invalid key ID**

→ wrong GPG format or wrong key selected.
The `signing.keyId` also requires your last 8 characters.

**❌ Public key not found**

→ forgot to upload GPG key:

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY
```

## Final result

After all this:

- Your SDK/lib is live on Maven Central
- Installable via Gradle globally
- Signed and validated
- Production-ready