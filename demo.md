# Demo: Migrating a Node.js Application to Chainguard Containers

In this demo, we'll migrate a Dockerfile for a Node.js command line application to Chainguard Containers. Before and after the migration, we'll scan our build for known vulnerabilities (CVEs) using [Grype](https://github.com/anchore/grype), a popular container image scanner.

We'll be moving through the following steps:

1. Install required software (Docker and Grype).
2. Create a sample CLI application using a Dockerfile building on the [official Node image](https://hub.docker.com/_/node/).
3. Scan the image with Grype to find out how many CVEs are present.
4. Migrate the Dockerfile to use the [Node Chainguard Container](https://images.chainguard.dev/directory/image/node/overview) as the base image.
5. Run the scanner again to see if the CVE situation has improved. 😇

After the demo, we'll share some resources for those who want to dig a little deeper. Let's jump in!

## Prerequisites

To follow this tutorial, you will need the following:

- Access to a UNIX-like terminal environment, such as the terminal on Mac OS or Linux or Windows Subsystem for Linux.
- A working installation of [Docker Engine](https://docs.docker.com/engine/install/) or [Docker Desktop](https://docs.docker.com/desktop/).
- The [Grype scanner](https://github.com/anchore/grype#installation).

On most systems, Grype can be installed using the following command:

```sh
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /usr/local/bin
```

## Dockerfile Build Using the Official Node Image

First, let's create a Dockerfile that uses the official Node image as a base image. The application will print the text `Hello, Linky! 🐙` to the console.

Using your terminal, create a project folder for our build and change the working directory to that folder:

```sh
mkdir -p ~/node-hello && cd $_
```

Now let's create a JavaScript file using a text editor. In this example, we'll use Nano, a console text editor available by default on many systems.

```sh
nano hello.js
```

Enter the following code:

```js
console.log("Hello, Linky! 🐙");
```

To save the file, press `Control-x`, then `y`, then hit `Enter`. This will save the file as `hello.js` in the current directory and close Nano.

Next, we'll need a Dockerfile:

```sh
nano Dockerfile
```

Enter the following text in the `Dockerfile`:

```Dockerfile
FROM node
COPY hello.js hello.js
ENTRYPOINT ["node", "hello.js"]
```

Save this text (`Control-x`, `y`, `Enter`).

Now let's build the image:

```sh
docker build . -t node-hello
```

Finally, run the image:

```sh
docker run node-hello
```

You should see the text `Hello, Linky! 🐙` as output. We've now containerized a one-line Node.js application using the official Node image.

Before moving on, take a look at how big that image is:

```sh
docker image ls node-hello
```

On October 5, 2026, the official Node image came in at 1.25 GB. Keep that number in mind.

## Scanning for CVEs with Grype

Now we'll use the Grype scanner to check this newly created Node image for vulnerabilities. For this exercise, we'll assume you have [installed Grype](https://github.com/anchore/grype#installation) on your host machine, as it's a useful tool to have in your vulnerability management toolbox. Chainguard also maintains a [Grype container image](https://images.chainguard.dev/directory/image/grype/overview) if you'd rather not install it.

Run the following command to scan the `node-hello` image with Grype:

```sh
grype node-hello
```

On running this command, you should see dynamic output from Grype as the image is cataloged and scanned. After some time, you should receive a large amount of output. Output from Grype is divided into two parts: an initial overview and an itemized list of specific vulnerabilities. Let's first take a look at the initial overview — you may need to scroll up to find it.

```
 ✔ Scanned for vulnerabilities     [1942 vulnerability matches]
   ├── by severity: 57 critical, 345 high, 433 medium, 135 low, 967 negligible (5 unknown)
   └── by status:   127 fixed, 1815 not-fixed
```

Those are the numbers from October 5, 2026. Yours will differ a little, because both the image and the vulnerability database change daily, but the shape will be the same: a one-line script riding on top of nearly two thousand vulnerability matches.

Arguably the most important line in this output is the breakdown of vulnerabilities by severity. CVEs are assigned numerical scores according to the [Common Vulnerability Scoring System (CVSS)](https://nvd.nist.gov/vuln-metrics/cvss). These scores correspond to four categories:

- Critical (9.0-10.0)
- High (7.0-8.9)
- Medium (4.0-6.9)
- Low (0.1-3.9)

The official Node image is built on a full Debian base, so almost everything Grype finds has nothing to do with Node itself. Of the 1,942 matches above, 1,931 were Debian operating system packages and 11 were npm packages. The operating system came along for the ride, and it brought its vulnerabilities with it.

Another line to look out for is how many of the CVEs have been fixed in a release:

```
   └── by status:   127 fixed, 1815 not-fixed
```

If you're seeing CVEs with a "fixed" status, that indicates that updating to a new version of the affected package will resolve the issue. The more often an image is rebuilt with updated packages, the fewer fixed CVEs will be found.

Grype also itemizes each CVE found during the scan. This constitutes the second half of our output. Let's rerun the scan to look just at the CVEs that have critical status:

```sh
grype node-hello | grep -i critical
```

On October 5, 2026, that produced 57 rows covering 19 distinct critical CVEs. Here are the first few:

```
libmariadb-dev                1:11.8.6-0+deb13u1    (won't fix)       deb   CVE-2026-49261   Critical
libmariadb-dev-compat         1:11.8.6-0+deb13u1    (won't fix)       deb   CVE-2026-49261   Critical
libmariadb3                   1:11.8.6-0+deb13u1    (won't fix)       deb   CVE-2026-49261   Critical
mariadb-common                1:11.8.6-0+deb13u1    (won't fix)       deb   CVE-2026-49261   Critical
libunbound8                   1.22.0-2+deb13u3      1.26.1-0+deb13u1  deb   CVE-2026-81642   Critical
curl                          8.14.1-2+deb13u5      (won't fix)       deb   CVE-2026-19931   Critical
libcurl4t64                   8.14.1-2+deb13u5      (won't fix)       deb   CVE-2026-19931   Critical
libopenexr-3-1-30             3.1.13-2                                deb   CVE-2026-42216   Critical
```

Notice two things. First, the same CVE shows up several times, once for each package built from the same source. Second, most of the critical findings are marked `(won't fix)`: the distribution has decided not to patch them in this release, so no amount of `apt upgrade` will make them go away. A MariaDB client library and an image-format library are in your Node container whether you wanted them or not.

If you're resolving CVEs manually, this output is your starting point. Generally, you would look up the CVE in the [National Vulnerability Database](https://nvd.nist.gov/), which collects information on affected packages and systems, disclosures, and additional resources such as advisories and potential mitigations. For packages marked `(won't fix)`, you will need to either determine that the CVE isn't relevant to your use case, remove the affected package, or manually mitigate the CVE.

## Migrating to Chainguard Containers

We've seen that using the official Node image as a base image for our application leads to a build with many known vulnerabilities. Let's switch to the Node Chainguard Container as our base image by changing our `Dockerfile`.

```sh
nano Dockerfile
```

Edit the `FROM` line to pull from Chainguard's public Node image:

```
FROM cgr.dev/chainguard/node
```

Your updated `Dockerfile` should look as follows:

```Dockerfile
FROM cgr.dev/chainguard/node
COPY hello.js hello.js
ENTRYPOINT ["node", "hello.js"]
```

Two things changed under the hood even though the Dockerfile barely did. The Chainguard image sets its working directory to `/app`, so `hello.js` lands there instead of at the filesystem root, and the container runs as the non-root `node` user instead of `root`. Neither matters for a one-line script, but they're worth knowing when you migrate a real application.

Build the Chainguard version of the image, tagging it `chainguard-hello`:

```sh
docker build . -t chainguard-hello
```

Now run the image:

```sh
docker run chainguard-hello
```

You should see output as before: `Hello, Linky! 🐙`. Check the size:

```sh
docker image ls chainguard-hello
```

On October 5, 2026, the Chainguard build was 178 MB, about one seventh the size of the official image. Smaller means less software, and less software means fewer places for CVEs to live. Let's confirm that by scanning the new build:

```sh
grype chainguard-hello
```

```
 ✔ Scanned for vulnerabilities     [55 vulnerability matches]
   ├── by severity: 1 critical, 12 high, 22 medium, 6 low, 0 negligible (14 unknown)
   └── by status:   49 fixed, 6 not-fixed
```

From 1,942 matches to 55, and from 57 criticals to 1. The Chainguard image doesn't ship a Debian userland, so the MariaDB, curl, and OpenEXR findings are simply not there. What remains is in Node's own toolchain: `npm` and the packages bundled inside it, `node-gyp`, and `zlib`.

```sh
grype chainguard-hello | head -8
```

```
NAME                  INSTALLED  FIXED IN               TYPE  VULNERABILITY        SEVERITY
zlib                  1.3.2-r5   1.3.2.1_rc20260601-r0  apk   CVE-2026-85091       High
http-cache-semantics  4.2.0                             npm   GHSA-ch52-4w7c-c8xp  High
undici                6.28.0     6.28.1                 npm   GHSA-rfgv-xxqx-mfg5  High
undici                8.10.0     8.10.2                 npm   GHSA-rfgv-xxqx-mfg5  High
node-gyp              13.0.2-r0  13.0.2-r1              apk   CVE-2026-85014       High
brace-expansion       5.0.9      5.0.10                 npm   GHSA-6j4f-fj2g-mc7p  High
npm-12                12.0.2-r2  12.1.0-r2              apk   CVE-2026-102276      High
```

Look at the `FIXED IN` column this time. On the official image it mostly read `(won't fix)`. Here, 49 of the 55 findings already name the version that fixes them, which means they are in flight: the image is rebuilt whenever an upstream dependency changes, so a scan a few days later will usually show these gone and a scan on a quiet day will show nothing at all. The remaining handful marked `Unknown` are CVEs still under investigation that may or may not be assigned a severity later.

That's the demo. Same one-line application, same three-line Dockerfile, one changed `FROM`, and the vulnerability count fell by about 97 percent. 😎

## Resources

- [Overview of Chainguard Containers](https://edu.chainguard.dev/chainguard/containers/overview/)
- [Getting Started with the Node Chainguard Container](https://edu.chainguard.dev/chainguard/containers/getting-started/languages-and-runtimes/node/)
- [How to Port a Sample Application to Chainguard Containers](https://edu.chainguard.dev/chainguard/containers/migration/porting-apps-to-chainguard/)
- [Node Chainguard Container in the Containers Directory](https://images.chainguard.dev/directory/image/node/overview)
- [Blog Post: Migrating a Node.js Application to Chainguard Images](https://www.chainguard.dev/unchained/migrating-a-node-js-application-to-chainguard-images)
