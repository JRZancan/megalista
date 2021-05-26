# Megalista

Sample integration code for onboarding offline/CRM data from BigQuery as custom audiences or offline conversions in Google Ads, Google Analytics 360, Google Display & Video 360 and Google Campaign Manager.

**Disclaimer:** This is not an officially supported Google product.

## Supported integrations
- **Google Ads**
  - Contact Info **Customer Match** (email, phone, address) [[details]](https://support.google.com/google-ads/answer/6379332?&ref_topic=6296507)
  - Id Based **Customer Match** (device Id, user id)
  - Offline Conversions through **gclid** [[details]](https://support.google.com/google-ads/answer/2998031?)
  - Store Sales Direct **(SSD)** conversions [[details]](https://support.google.com/google-ads/answer/9995886?hl=en)

- **Google Analytics (Universal analytics)**
  - Custom segments through **Data Import** [[details]](https://support.google.com/analytics/answer/3191589?hl=en)
  - Measurement Protocol [[details]](https://developers.google.com/analytics/devguides/collection/protocol/v1#:~:text=Measurement%20Protocol%20Overview%20bookmark_border&text=The%20Google%20Analytics%20Measurement%20Protocol,directly%20to%20Google%20Analytics%20servers.)

- **Campaign Manager**
  - Offline Conversions API **(user id, device id, match id, gclid, dclid)** [[details]](https://developers.google.com/doubleclick-advertisers/guides/conversions_upload)

- **Google Analytics 4**
  - Measurement protocol (Web + App) [[details]](https://developers.google.com/analytics/devguides/collection/protocol/ga4)

- **Appsflyer**
  - S2S Offline events API (conversion upload), to be used for audience creation and in-app events with Google Ads and DV360 [[details]](https://support.appsflyer.com/hc/en-us/articles/207034486-API-de-eventos-de-servidor-para-servidor-S2S-mobile-para-mobile)

## How does it work
Megalista was design to separate the configuration of conversion/audience upload rules from the engine, giving more freedom for non-technical teams (i.e. Media and Business Inteligence) to setup multiple upload rules on their own.

The solution consists of #1 a configuration file (either Google Sheet or JSON file) in which all rules are defined by mapping a data source (BigQuery Table) to a destination (data upload endpoint) and #2, an Apache Beam workflow running on Google Dataflow, scheduled to upload the data in batch mode.

## Prerequisites

### Google Cloud Services
- **Google Cloud Platform** account
  - **Billing** enabled
  - **BigQuery** enabled
  - **Dataflow** enabled
  - **Cloud storage** enabled
  - **Cloud scheduler** enabled
- At least one of:
  - **Google Ads** API Access
  - **Campaign Manager** API Access
  - **Google Analytics** API Access
- **Python3**
- **Google Cloud SDK**

### Access Requirements
Those are the minimum roles necessary to deploy Megalista:
- OAuth Config Editor
- BigQuery User
- BigQuery Job User
- BigQuery Data Viewer
- Cloud Scheduler Admin
- Storage Admin
- Dataflow Admin
- Service Account Admin
- Logs Viewer
- Service Consumer

### APIs
Required APIs will depend on upload endpoints in use. We recomend you to enable all of them:
- Google Sheets (required if using Sheets configuration) [[link]](https://console.cloud.google.com/apis/library/sheets.googleapis.com)
- Google Analytics [[link]](https://console.cloud.google.com/apis/library/analytics.googleapis.com)
- Google Analytics Reporting [[link]](https://console.cloud.google.com/apis/library/analyticsreporting.googleapis.com)
- Google Ads [[link]](https://console.cloud.google.com/apis/library/googleads.googleapis.com)
- Campaign Manager [[link]](https://console.cloud.google.com/apis/library/dfareporting.googleapis.com)


## Installation

### Configure Megalista
Megalista can be configured via either Google Sheets or a JSON file. Expected data schemas (Sources) and metadata (Destinations) for each use case defined in [the Megalista Wiki](https://github.com/google/megalista/wiki).

To configure using Google Sheets:
 - Make a copy of the [Sheets template](https://docs.google.com/spreadsheets/d/1rP9x93h5CVu2IdUjP6yJ8yYEMcb9J_WPx2sBEEpLMJI)
 - In the "Intro" sheet provide Account IDs for Google Ads, Analytics, CM360, etc.
 - Configure source/input data in the "Sources" sheet
 - Configure destinations in the "Destinations" sheet
  + The "Help" sheet has a chart to document the required metadata for each use case.
 - Configure connections from Sources to Destinations in the "Connect" sheet
 - Make note of the Sheet's ID in the URL: `docs.google.com/spreadsheets/d/`**EXAMPLEIDHERE**`/edit`

 To configure using JSON:
 - Make a copy of the [JSON template](https://github.com/google/megalista/tree/main/cloud_config/configuration_sample.json)
 - Provide Account IDs for Google Ads, Analytics, CM360, etc.
 - Configure Sources and Destinations, separating entries with commas
 - Connect Sources to Destinations in the "Connections" field
 - Save and upload this JSON file to a Google Cloud Storage bucket (default bucket options are fine)
 - Make note of the files's Authenticated URL in the Cloud Storage UI: `storage.cloud.google.com/`**mybucketname/myconfig.json**

### Creating required access tokens
To access campaigns and user lists on Google's platforms, this dataflow will need OAuth tokens for a account that can authenticate in those systems.

In order to create it, follow these steps:
 - Access GCP console
 - Go to the **API & Services** section on the top-left menu.
 - On the **OAuth Consent Screen** and configure an *Application name*
 - Then, go to the **Credentials** and create an *OAuth client Id* with Application type set as *Desktop App*
 - This will generate a *Client Id* and a *Client secret*
 - Run the **generate_megalist_token.sh** script in this folder providing these two values and follow the instructions
   - Sample: `./generate_megalist_token.sh client_id client_secret`
 - This will generate the *Access Token* and the *Refresh token*

### Creating a bucket on Cloud Storage
This bucket will hold the deployed code for this solution. To create it, navigate to the *Storage* link on the top-left menu on GCP and click on *Create bucket*. You can use Regional location and Standard data type for this bucket.

## Running Megalista

We recommend first running it locally and make sure that everything works.
Make some sample tables on BigQuery for one of the uploaders and make sure that the data is getting correctly to the destination.
After that is done, upload the Dataflow template to GCP and try running it manually via the UI to make sure it works.
Lastly, configure the Cloud Scheduler to run Megalista in the frequency desired and you'll have a fully functional data integration pipeline.

### Running locally
Only set one configuration parameter (either setup_sheet_id or setup_json_url)
```bash
python3 megalist_dataflow/main.py \
  --runner DirectRunner \
  --developer_token ${GOOGLE_ADS_DEVELOPER_TOKEN} \
  --setup_sheet_id ${CONFIGURATION_SHEET_ID} \
  --setup_json_url ${CONFIGURATION_JSON_URL} \
  --refresh_token ${REFRESH_TOKEN} \
  --access_token ${ACCESS_TOKEN} \
  --client_id ${CLIENT_ID} \
  --client_secret ${CLIENT_SECRET} \
  --project ${GCP_PROJECT_ID} \
  --region us-central1 \
  --temp_location gs://{$GCS_BUCKET}/tmp
```

### Deploying Pipeline
To deploy the full Megalista pipeline, use the following command from the root folder:
`./terraform_deploy.sh`

#### Manually executing pipeline using Dataflow UI
To execute the pipeline, use the following steps:
- Go to **Dataflow** on GCP console
- Click on *Create job from template*
- On the template selection dropdown, select *Custom template*
- Find the *megalist* file on the bucket you've created, on the templates folder
- Fill in the parameters required and execute

### Scheduling pipeline
To schedule daily/hourly runs, go to **Cloud Scheduler**:
- Click on *create job*
- Add a name and frequency as desired
- For *target* set as HTTP
- Configure a *POST* for url: https://dataflow.googleapis.com/v1b3/projects/${YOUR_PROJECT_ID}/locations/${LOCATION}/templates:launch?gcsPath=gs://${BUCKET_NAME}/templates/megalist, replacing the params with the actual values
- For a sample on the *body* of the request, check **cloud_config/scheduler_sample.json**
- Add OAuth Headers
- Scope: https://www.googleapis.com/auth/cloud-platform

#### Creating a Service Account
It's recommended to create a new Service Account to be used with the Cloud Scheduler
- Go to IAM & Admin > Service Accounts
- Create a new Service Account with the following roles:
    - Cloud Dataflow Service Agent
    - Dataflow Admin
    - Storage Objects Viewer


## Usage
Every upload method expects as source a BigQuery data with specific fields, in addition to specific configuration metadata. For details on how to setup your upload routines, refer to the [Megalista Wiki](https://github.com/google/megalista/wiki) or the [Megalista user guide](https://github.com/google/megalista/blob/main/documentation/Megalista%20-%20Technical%20User%20Guide%20-%20EXTERNAL.pdf).
