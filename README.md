# Access token server: Looker extension app + auth w/ Google service account

## Get Google OAuth access tokens using App Engine's default service account

This project is primarily provided to support users of the [Looker BQML Accelerator extension](https://github.com/looker-open-source/app-bqml-accelerator) who would like to use a service account to authenticate with BigQuery (rather than OAuth2 end-user credentials). The intent here is to provide a simple path to success for users who want to get started right away using the Looker BQML Accelerator app with a Google service account, but who may not be developers and/or who need technical guidance/bootstrapping to get such a server set up.

In order for the BQML Accelerator app to obtain access tokens with which to make its requests to the BigQuery API, users will first need to stand up this simple backend service in GCP App Engine, and then complete a few other straightfoward configuration steps so that the Looker extension can use this service. Detailed, step-by-step guidelines are provided here to help you get up and running quickly.

## Why do I need this?

Authentication with a Google service account has some different requirements than authenticating with end-user credentials. Mainly, we need to be able to create and sign JWTs in order to request access tokens when using a service account (this is not needed when using end-user creds). Creating and signing JWTs should be done securely on a server (not, say, in the browser, where the Looker extension app runs). Thus, a server-side component is needed, and this project will help you quickly set up a basic, adequate service for this need.

### OK, but why this GCP App Engine approach?

This project is set up to deploy only to GCP App Engine and to only use its attached default service account to get access tokens. There are certainly many, many other ways of setting up a service such as this, so why did we take this particular approach? We decided to prescribe the use of App Engine and its attached default service account so that the process takes fairly few steps, requires minimal technical knowledge and, most importantly, does not lead the user to directly handle any service account keys in any way (and thereby avoid the potential of introducing security risks).

## How do I set this up?

There are three basic steps/parts:

1. Configure and deploy the access token server to App Engine
2. Adjust the App Engine default service account's roles and create firewall rules
3. Install and configure the Looker BQML extension app

Please see [the detailed setup guide here](docs/README-SETUP.md)

## Acknowledgements / Related Resource

Significant credit and thanks are due to the contributors to [this excellent open-source Looker SDK source code example](https://github.com/looker-open-source/sdk-codegen/tree/main/examples/access-token-server), which provided the foundation for this project's core functionality. Please take a look there for more great starting points if you wish to build out a more complex implementation (such as one that can use user-managed service account keys and deploy to environments other than App Engine).
