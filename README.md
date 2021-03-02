# iOS Signer Service

> A self-hosted, cross-platform service to sign iOS apps using any CI as a builder

## Introduction

There are many reasons to install apps outside the App Store. Unfortunately, this process is severely hindered by Apple and unrealistic for the average user. You need a better way to get things done.

Welcome `ios-signer-service` - a self-hosted, cross-platform service to sign iOS apps and install them on your device, all without a computer.

The setup consists of two parts - this web service, which can run on any computer, and a macOS builder, which this service controls to perform the actual signing. The builder integration is minimal, so any (free) API-enabled Continuous Integration (CI), such as GitHub, can be used.

**NOTICE**: You must have a valid **signing certificate** and **provisioning profile**. This project does not circumvent any protections.

## Features

- No jailbreak required
- All iOS versions supported
- No computer required (apart from server to host the service)
- Works with any CI provider
- Minimalistic, mobile-friendly web interface
- Upload unsigned apps, download signed apps
- Install signed apps straight to your iOS device via [OTA](https://medium.com/@adrianstanecki/distributing-and-installing-non-market-ipa-application-over-the-air-ota-2e65f5ea4a46)
- Choose from multiple signing profiles
- Configure the signing process
- Periodic old file cleanup

## Screenshots

<table>
<tr>
    <th>Mobile</th>
    <th>Desktop</th>
</tr>
<tr>
    <td>
        <img src="img/3.png"/>
        <img src="img/4.png"/>
    </td>
    <td>
        <img src="img/1.png"/>
        <img src="img/2.png"/>
    </td>
</tr>
</table>

## Installation

### Requirements

- Any server; even a Raspberry Pi will work
- Reverse proxy with valid HTTPS (self-signed will NOT work with OTA)
- Builder system, such as a CI, that:
  - Runs macOS
  - Supports workflow triggers via API
- Valid code signing profile:
  - Certificate (`.p12` file) and password
  - Provisioning profile (`.mobileprovision` file)

### Web Service

`ios-signer-service` (this project) is a web service that you install on any server. The service exposes a web interface which allows the user to upload unsigned app files and have them signed using any of the configured signing profiles. The service offloads the signing process to a dedicate macOS builder, more on which in the next section.

The easiest way to install the service is using the [Docker image](https://hub.docker.com/r/signtools/ios-signer-service). When you run the program for the first time, it will exit immediately and generate a configuration file. Make sure you set it appropriately.

Inside the `save_dir` directory ("data" by default), you need to add at least one code signing profile. The structure is as follows:

```
data
|____profiles
| |____PROFILE_ID              # any unique string that you want
| | |____cert.p12              # the signing certificate
| | |____pass.txt              # the signing certificate's password
| | |____name.txt              # a name to show in the web interface
| | |____prov.mobileprovision  # the signing provisioning profile
| |____OTHER_PROFILE_ID
| | |____...
```

By default, `ios-signer-service` does not offer encryption (HTTPS) or global authentication. This is a huge security issue, and OTA installations will simply not work! To make things work, you need to run a reverse-proxy like nginx, which wraps the service with HTTPS and authentication. You must leave a few endpoints non-authenticated, since they are used by OTA and builder. Don't worry, they are secured by long ids and the workflow key:

```
/apps/:id/
/jobs
/jobs/:id
```

(where `:id` is a wildcard parameter)

When an app is uploaded to the service for signing, a signing job is generated and stored in memory. The service then triggers the builder using the configured workflow trigger API. The builder will query the available jobs from the service using the `/jobs` endpoint from above, and download the most recent job's data. The data is a simple TAR file which contains all the necessary signing files. When the builder is finished, it will upload the signed file to the service using a "return id" found within the archive.

### Builder

As mentioned before, `ios-signer-service` offloads the signing process to a dedicated macOS builder. This process is necessary because signing is only officially supported on a macOS system. While third-party cross-platform alternatives exist, they are not as stable or quick to update as the official solution.

A free and simple implementation of a builder can be found in [ios-signer-ci](https://github.com/SignTools/ios-signer-ci). It demonstrates how to use popular CI services. To host your own, simply fork the repo and follow its README.
