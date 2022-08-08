## Prerequisites

### 1. Assumptions about your GCP project: App Engine is not already in use

This guide assumes that in your GCP project (the one in which your BigQuery instance resides), the App Engine environment is not already in use. If your project already has some App Engine app using it, then these exact steps will not work for you, and you'll want to work with your GCP admin and/or a developer to deploy your access token server in a different fashion. It's possible, for instance, to create a new project in your GCP org for this App Engine app and then enable it to access your BigQuery instance in the other project (by creating a cross-project service account), but that is beyond the scope of this guide.

See [GCP's Overview of App Engine](https://cloud.google.com/appengine/docs/standard/nodejs/an-overview-of-app-engine) docs for more details.

### 2. Check your GCP permissions

In order to be successful, you will need to have (or get the assistance of someone who has) the necessary permissions in your GCP project (the same project that your BigQuery instance is in) in order to:

1. Deploy to App Engine
1. Make updates to a service account's roles

If you are an `owner` of your GCP project, you'll be all set. If not, you can still have success if you are able to have your GCP administrator grant you the following roles (or assist you in performing the steps outlined in the next section below):

1. `App Engine Admin` (`roles/appengine.appAdmin`) and `Service Account User` (`roles/iam.serviceAccountUser`) on the App Engine default service account
1. In order to deploy the app using `gcloud` cli tool (which you will use inside Cloud Shell): `Storage Object Admin` (`roles/storage.objectAdmin`) and `Cloud Build Editor` (`roles/cloudbuild.builds.editor`)
1. In order to update the default service account after you deploy: `Service Account Admin` (`roles/iam.serviceAccountAdmin`)

For further details regarding roles that grant access to App Engine, [see these GCP docs](https://cloud.google.com/appengine/docs/standard/nodejs/roles)

Again, you will want to ensure that your use of App Engine and its default service account does not conflict with other related resources in your GCP project.

### 3. Use GCP Cloud Shell to deploy

You don't need to be a developer to set this up. If you're comfortable using [GCP Cloud Shell](https://cloud.google.com/shell) (or willing to learn the basics of this useful in-browser tool), you'll be able to get up and running using this guide. See [Cloud Shell tutorial](https://cloud.google.com/shell/docs/deploy-app-engine-app) for an overview first, if you'd like.

---

## Set-up Steps

The following steps are written assuming that the person executing them has met the above prerequisites. If you attempt any of the below and encounter GCP permission errors, you'll need to contact your GCP admin for assistance.

### 1. Deploy access token server to GCP App Engine using Cloud Shell

- Browse directly to the Cloud Shell Editor to open a session: https://console.cloud.google.com/cloudshelleditor?cloudshell=true . **Make sure you are in the correct GCP project** (the same project that your BigQuery instance is in).
- Click on the "Open Cloud Shell Editor" button. This will load a new Cloud Shell VM and will look like a developer IDE. You will see a button at the top labeled "Open Terminal". If you click that, you'll be taken to a Terminal window. Click "Open Editor" to go back to the Editor. You will type the following commands in either the Terminal or Editor as described below.
  1. First you need to clone the [the access token server repo](https://github.com/looker-open-source/access-token-server) by entering the following command in Cloud Shell Terminal: `git clone https://github.com/looker-open-source/access-token-server`
  2. Now you need to set one environment variable that our app needs. Go back to Cloud Shell Editor, and you will now see a `looker-ext-access-token-server` folder in the file tree on the left. Inside this folder, there is a file called `app.yaml`. Open this file and set the value for the required environment variable `LOOKERSDK_BASE_URL` on Line 4. The value should be the URL for your Looker instance, such as `https://your_company.looker.com`. Make sure there is a space between the colon `:` and the start of your URL (notice the IDE color-coding changes to help you). Cloud Shell should automatically save your changes, but make sure this change is saved before moving on.
  3. Now go back to Terminal and deploy your app to App Engine by typing this command: `cd access-token-server/ && gcloud app deploy` (this command first moves into the directory where our `app.yaml` file is and then tells gcloud to deploy our app to the default service in our project. You will be guided through steps, including authorizing gcloud to deploy on your behalf, and then to confirm the details of what you're deploying. You should be deploying to the `default` service in your project. Type 'Y' if all looks right. It may take a few minutes to complete, especially if your App Engine default service is being created for the first time now.
  4. Once the deploy has successfully completed, you will see a line printed in the Terminal: `Deployed service [default] to [https://{YOUR_GCP_PROJECT}.uc.r.appspot.com]`. You will need this URL later when you set Looker user attributes, so make note of it now.

### 2a. Adjust default service account's role

#### **!! DO NOT SKIP THIS !!**

**NOTE:** This step can only be performed by a GCP principal (user) with `Service Account Admin` role (or higher privileges).

This crucial step is advised as a best practice to avoid creating security risks in your project. [See GCP docs for details on automatic role grants](https://cloud.google.com/iam/docs/best-practices-for-securing-service-accounts#automatic-role-grants).

- Go to [IAM & Admin console](https://console.cloud.google.com/iam-admin/iam), select your project, and find the default service account that was just created for your new app when you deployed it to App Engine:

  - Principal (email address) will be: `{YOUR_GCP_PROJECT_ID}`@appspot.gserviceaccount.com
  - Name will be: "App Engine default service account"
    - **Remove the `Editor` role**
    - Add only the roles necessary for Looker BQML Accelerator extension app to perform its actions against BigQuery:
      - Add `BigQuery Data Editor` role
      - Add `BigQuery Job User` role

  You can now spot-check your access token server by sending a request to the new `POST /access_token` endpoint. Here's a cURL example (replace the three `{TEMPLATE}` placeholders with valid values):

  ```
    curl --request POST \
      --url {YOUR_APP_ENGINE_URL}/access_token \
      --header 'Content-Type: application/json' \
      --data '{
    	"client_id": {YOUR_LOOKER_CLIENT_ID},
    	"client_secret": {YOUR_LOOKER_CLIENT_SECRET},
    	"scope": "https://www.googleapis.com/auth/bigquery"
    }'
  ```

  If you provide valid Looker client ID and secret and the correct app engine URL, you should receive a 200 OK with a response payload containing an object that has `access_token` and `expiry_date` fields

### 2b. Set Firewall Rules on App Engine app (technically optional, but highly recommended!)

You can (and should) lock down access to your access token server by setting some simple firewall rules on your App Engine service. Rather than leaving your access token server open to all internet traffic, you can restrict it so that effectively only requests coming from your Looker extension app will be accepted.

- Find your Looker instance's public IP addresses: https://docs.looker.com/admin-options/database/connections
- Browse to the App Engine Firewall console here: https://console.cloud.google.com/appengine/firewall
  - Set one firewall rule for each IP. See [GCP docs regarding App Engine firewalls](https://cloud.google.com/appengine/docs/standard/nodejs/creating-firewalls). Ensure these have higher priority (1, 2, 3...) than the default rule, which has lowest priority by default.
  - Change the "default" rule from "`Allow *`" to "`Deny *`" (so that it will deny all other traffic after allowing in requests from your Looker instance IPs)

### 3. Configure the Looker extension app

1. Now set the three additional user attributes required for the service account auth strategy:

   - `looker_client_id` and `looker_client_secret`
     - These refer to Looker API3 credentials ( [see this Looker API auth documentation](https://docs.looker.com/reference/api-and-integration/api-auth), and ["Edit API3 Keys" section of this Looker doc](https://docs.looker.com/admin-options/settings/users))
     - These credentials are passed from the BQML Accelerator extension to your access token server in order to validate that the token request is coming from someone with valid Looker API creds for your Looker instance. **See documentation in the access token server repo for more details**.
       - You can set default values using one set of API3 credentials (a single Looker admin) for all your users or a group of users, or you can set individual user values so each of your users is using their own API keys.
   - `access_token_server_endpoint`

     - This is the endpoint for your App Engine app, which the extension app will call to request a token. **REMEMBER: must end in `/access_token`!** So something like: `https://{YOUR_GCP_PROJECT_ID}.uc.r.appspot.com/access_token`
     - **NOTE:** If this user attribute is not properly set, the BQML Accelerator app will not be able to use service account authentication and will attempt to fall back on OAuth2 implicit flow instead (using end-user credentials rather than service account)

     **NOTE:** All of these user attributes **must be namespaced/scoped for the extension app** ([see Looker docs regarding scoped user attributes](https://docs.looker.com/data-modeling/extension-framework/js-r-extension-examples#user_attributes)), so make sure that the **attribute names** are prepended with `your_project_name_your_project_name_`.

     For example, if your Looker project name is `bqml-accelerator`, then the name for the `looker_client_id` user attribute should be: `bqml_accelerator_bqml_accelerator_looker_client_id`)

2. Make sure these three scoped user attributes are in your extension app's manifest file as **`scoped_user_attributes`** entitlements
   (see example below)
3. Finally, also set the access token server URL in the manifest file as an `external_api_urls` entitlement (so that the extension app will be able to call your access token server to get tokens) (see example below)
