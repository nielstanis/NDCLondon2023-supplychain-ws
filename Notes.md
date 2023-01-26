# Workshop

- What's the problem space?
- Today's agenda?
- The commands used in this workshop will be running on either MacOSX and/or Linux (WSL)

# Introducing DocGenerator Project

A library called `docgenerator` that has been published on feedz.io can be used to process documents. 

## Lab 1 Setup repo + build

In order to start create a fork of the `https://github.com/nielstanis/docgenerator` repository.

Make sure you're enabling Actions and execute the current .NET build available to generate the output. 

## Git Commit Signing

It's a real good idea to start git commit signing, resulting in an additional factor that can be verified. If in case of credential theft it can be as an extra indicator if something bad has happened. 

We're not going to add it to our current workflow because it contains a lot of loop holes and constrains and it will probably fail. 

- https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits 
- https://www.hanselman.com/blog/how-to-setup-signed-git-commits-with-a-yubikey-neo-and-gpg-and-keybase-on-windows 
- GitSign with Sigstore - https://blog.sigstore.dev/introducing-gitsign-9fd3f1b682aa

# Reproducible Builds

## Lab 2

- Create new console with the following CLI call : `dotnet new console -n ConsoleApp`
- Rename the folder `ConsoleApp` to `ConsoleApp2`
- Execute `dotnet new console -n ConsoleApp` again that will create a new folder
- Now build both of the projects
  - `dotnet build ConsoleApp\ConsoleApp.csproj`
  - `dotnet build ConsoleApp2\ConsoleApp.csproj`
- What will the output binaries look like? 
  - `diff ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll`
- Maybe you can decompile (e.g. JetBrains Rider) and look into what it looks like?
- Run string on the binaries and create text files to compare with e.g. VS Code or any other preferred tool
  - `strings ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll >> ConsoleAppStrings`
  - `strings ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll >> ConsoleApp2Strings`
  - `code -r --diff ConsoleAppStrings ConsoleApp2Strings`
- Alter both `.csproj` files and add `<DebugType>None</DebugType>` to it

## Roslyn Compiler Deterministic

Based on given `inputs` the produced `outputs` should be the same. This has been done since Roslyn v1.1

https://github.com/dotnet/roslyn/blob/master/docs/compilers/Deterministic%20Inputs.md

## Lab 3 - Dotnet.Reproducible

Now let us add two packages to our project that will help out with setting certain project settings that can be found here:

https://github.com/dotnet/reproducible-builds/blob/main/README.md

Add `Dotnet.Reproducible` and `Dotnet.Reproducible.Isolated` (SDK) and commit the changes initialing the build to produce a new artifact. 

## NuGet Proposed Design
   
https://github.com/dotnet/designs/blob/main/accepted/2020/reproducible-builds.md

https://github.com/dotnet/designs/pull/171 

# SBOM - Why Needed

## Lab 4 - CycloneDX

Now we're going to add CycloneDX to our build pipeline. The existing Action shared in the marketplace is not working properly, that's why we're going to add the dotnet cli library and use that. Take the following steps.

- Add `dotnet tool install --global CycloneDX` as next step with name `Install CylconeDx in dotnet cli` to our build after we've done our `dotnet test`.
- Add `dotnet CycloneDX docgenerator.csproj -j -o . ` as next step with name `CycloneDX .NET Generate SBOM`.
- We need to add the `bom.json` file to our stored outputs of pipeline, by creating multine files in `actions/upload-artifact` step

Your build definition will look similar to this: 

```yaml
name: .NET

on:
  workflow_dispatch:

jobs:
  build:

    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build -c Release -o .
    - name: Test
      run: dotnet test --no-build --verbosity normal
    - name: Install CylconeDx in dotnet cli
      run: dotnet tool install --global CycloneDX
    - name: CycloneDX .NET Generate SBOM
      run: dotnet CycloneDX docgenerator.csproj -j -o . 
    - name: Upload a Build Artifact
      uses: actions/upload-artifact@v3.1.0
      with:
          path: | 
              docgenerator.1.0.0.nupkg
              bom.json
          name: DocGenerator
          if-no-files-found: error
          retention-days: 5
```

Now commit+push the changes to GitHub, get pipeline to run and download `bom.json` and look into it's contents. 

Are there any vulnerabilties in the output? Consider checking `dotnet list package --vulnerable`. Did you see _all_ that's inside?

# SLSA Generator Provenance Generator

## Lab 5 - Provenance

We're going to use the default Provenance Generator for GitHub Actions. We need to make sure to do the following things to our build pipeline so it will look like the example output you find below. 

- Make sure permissions get added to the `runs-on` section that will allow `write` to `packages` and `id-token`. 
- Also we need to define `outputs` that will have the name of `hashes` and gets `${{ steps.hash.outputs.hashes }}`.
- Next up is going to generate sha256 hashes of the artifacts we want to tie to our generated provenance:
```yaml
- name: Generate hashes
    shell: bash
    id: hash
    run: |
    # sha256sum generates sha256 hash for all artifacts.
    # base64 -w0 encodes to base64 and outputs on a single line.
    # echo "::set-output name=hashes::$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)"
    echo "hashes=$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)" >> $GITHUB_OUTPUT
```
- The `hashes` are combined and set as output in base64 encoded string. 
- After that we're adding the provenance generation job which will look as follows:
```yaml
provenance:
needs: [build]
permissions:
    id-token: write # For signing.
    contents: write # For asset uploads.
    actions: read # For the entrypoint.
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@main
with:
    base64-subjects: "${{ needs.build.outputs.hashes }}"
    compile-generator: true
```
- This will generate the output and tie the previously generated hashses to it. It will under the hood also sign the artifact wih a project called SigStore which we'll touch later. 
- The output build pipline can be committed+pushed to our GitHub repo to start the pipeline and create arficats. For reference the pipeline should look following;

```yaml
name: .NET

on:
  workflow_dispatch:

jobs:
  build:

    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}

    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build -c Release -o .
    - name: Test
      run: dotnet test --no-build --verbosity normal
    - name: Install CylconeDx in dotnet cli
      run: dotnet tool install --global CycloneDX
    - name: CycloneDX .NET Generate SBOM
      run: dotnet CycloneDX docgenerator.csproj -j -o . 
    - name: Generate hashes
      shell: bash
      id: hash
      run: |
        # sha256sum generates sha256 hash for all artifacts.
        # base64 -w0 encodes to base64 and outputs on a single line.
        # echo "::set-output name=hashes::$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)"
        echo "hashes=$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)" >> $GITHUB_OUTPUT
    - name: Upload a Build Artifact
      uses: actions/upload-artifact@v3.1.0
      with:
          path: | 
              docgenerator.1.0.0.nupkg
              bom.json
          name: DocGenerator
          if-no-files-found: error
          retention-days: 5
  
  provenance:
    needs: [build]
    permissions:
      id-token: write # For signing.
      contents: write # For asset uploads.
      actions: read # For the entrypoint.
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@main
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      compile-generator: true

```

- Next up is to download the generated artifact, zip containg SBOM and NuGet and SLSA provenance and check out it's contents.
- Extract payload and decode from `multiple.intoto.json` by executing `cat multiple.intoto.jsonl | jq -r '.payload' | base64 -d > payload.json`
- Generate SHA256 has of both SBOM and NuGet Package and cross check: `shasum -a 256 docgenerator.1.0.0.nupkg bom.json`
- It also signs the generated provenance that can be verified against the public information on Sigstore system, let's check that later. 

# ASP.NET MVC that uses DocGenerator

## Lab 6 - Create ASP.NET MVC uses DocGenerator and runs on Docker

- Create new MVC site with name `docwebgen` by calling `dotnet new mvc -n docwebgen`.
- Add following `nuget.config` file to project:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
 <packageSources>
    <!--To inherit the global NuGet package sources remove the <clear/> line below -->
    <clear />
    <add key="docgen-packages" value="https://f.feedz.io/fennec/docgenerator/nuget/index.json" /> 
    <add key="nuget" value="https://api.nuget.org/v3/index.json" />
</packageSources>
</configuration>
```

- Create a `Dockerfile` that contains the following. 

```Dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
COPY nuget.config ./
RUN dotnet restore --configfile "./nuget.config"

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "docwebgen.dll"]
```
- Create a new `.gitignore` file by calling `dotnet new gitignore`.
- Initialize new Git repo `git init`.
- Add changes to it `git add .` and commit `git commit -m "Website" -S` (lose `-S` if your not signing the commit). 
- Create a new GitHub Repo and call it `DocWebGen` and push changes to that one. 
- We need to create a GitHub Actions pipeline that looks following, make sure to put in the right reponame. 

```yaml
name: container

on:
  push:
    tags:        
      - v1             # Push events to v1 tag
      - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
  workflow_dispatch:
    
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v1

      - name: Login to GitHub
        uses: docker/login-action@v1.9.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v2.7.0
        with:
          context: .
          push: true
          tags: ghcr.io/YOURREPOHERE/docgenweb:latest
```

# Docker image + Syft SBOM

Last year Docker released SBOM support that is based on a project caled `Syft` that will generate SBOM data of a container and it's contents.
We can add this pretty easily to our build pipeline based on the following lab

## Lab 7 Anchor Syft

```yaml
name: container

on:
    push:
    tags:        
        - v1             # Push events to v1 tag
        - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
    workflow_dispatch:

jobs:
build:
runs-on: ubuntu-latest
permissions:
    packages: write
    id-token: write

steps:
    - uses: actions/checkout@v1

    - name: Login to GitHub
      uses: docker/login-action@v1.9.0
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: build+push
      uses: docker/build-push-action@v2.7.0
      with:
        context: .
        push: true
        tags: ghcr.io/YOURREPOHERE/docgenweb:latest
        
    - uses: anchore/sbom-action@v0
      with:
        image: ghcr.io/YOURREPOHERE/docgenweb:latest

```

# Ubuntu Chiseled and Chainguard Wolfi

Consider moving to distro-less Linux distributions to reduce attack surface this can be achieved with Ubuntu Chiseled and Chainguards Wolfi. 

# Sigstore

Now we're going to use cosign to sign both a local file with keyless workflow and the container in our build. 

## Lab 8 Sign and validate local file keyless

In order to do this locally you need to get cosign installed:

https://edu.chainguard.dev/open-source/sigstore/cosign/how-to-install-cosign/ 

- `echo "Hello London" >> ndclondon.txt`
- `COSIGN_EXPERIMENTAL=1 cosign sign-blob --rekor-url https://rekor.sigstore.dev --oidc-issuer https://oauth2.sigstore.dev/auth ndclondon.txt`
- `rekor-cli get --log-index IDHERE --format json | jq > output.json`
- `cat output.json | jq -r '.Body.HashedRekordObj.signature.content' > ndclondon.txt.sig`
- `cat output.json | jq -r '.Body.HashedRekordObj.signature.publicKey.content' | base64 -d > pub.crt`
- `openssl x509 -noout -text -in pub.crt`
- `COSIGN_EXPERIMENTAL=1 cosign verify-blob --cert pub.crt --signature ndclondon.txt.sig ndclondon.txt`

## Lab 9 Cosign Container

Now we need to add both the sigining and verification with Cosign into our build pipeline. This is seen in the steps with `cosign sign` and `cosign verify` below. 

```yaml
name: container

on:
  push:
    tags:        
      - v1             # Push events to v1 tag
      - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
  workflow_dispatch:
    
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v1

      - name: Login to GitHub
        uses: docker/login-action@v1.9.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v2.7.0
        with:
          context: .
          push: true
          tags: ghcr.io/nielstanis/docgenweb:latest

      - uses: sigstore/cosign-installer@main

      - name: Sign the images
        run: |
          cosign sign \
            ghcr.io/YOURREPO/docgenweb:latest
        env:
          COSIGN_EXPERIMENTAL: 1

      - name: Verify the pushed tags
        run: cosign verify ghcr.io/YOURREPO/docgenweb:latest
        env:
          COSIGN_EXPERIMENTAL: 1
          
      - uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/YOURREPO/docgenweb:latest

```

We can get cosign (if installed locally) to verify as well, it also makes sense to look into the generated data by doing the following:

- `rekor-cli get --log-index IDREG --format json | jq > output.json`
- `cat output.json | jq -r '.Body.HashedRekordObj.signature.publicKey.content' | base64 -d > pub.crt`
- `openssl x509 -noout -text -in pub.crt`
- `COSIGN_EXPERIMENTAL=1 cosign verify ghcr.io/YOURREPOHERE/docgenweb:latest`

# Hacked now what?

## Lab 10

Maybe you should try to figure out how it got `Hacked` :D