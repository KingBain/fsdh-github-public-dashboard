# Databricks Dashboard Integration Setup

This project dynamically injects a short-lived Databricks token into a static site during the GitHub Actions deployment.

To make this work, you need:

1. A Databricks Service Principal
2. A published Databricks dashboard
3. GitHub repository environment, variables and secrets configured

## Demo Site
<img width="1188" height="2187" alt="image" src="https://github.com/user-attachments/assets/cc3d2738-464c-402e-b7f6-a39fca3d281d" />

---

## 1. Create a Databricks Service Principal

1. Go to your Databricks workspace
<img width="863" height="296" alt="image" src="https://github.com/user-attachments/assets/85a6479b-71fa-4b8b-b94f-b097b88ba2cc" />

2. Navigate to:
   * **Settings → Workspace admin → Identity and Access → Service Principals**
4. Create a new service principal
<img width="1018" height="687" alt="image" src="https://github.com/user-attachments/assets/9972174e-ce59-434a-9f8f-922a9efeb556" />
Select none of the "entitlements"



5. Create an OAuth "Secret" Private Key
<img width="986" height="573" alt="image" src="https://github.com/user-attachments/assets/4b0a6019-48a5-47e8-b583-73a0a859371a" />
<img width="1061" height="609" alt="image" src="https://github.com/user-attachments/assets/9908e121-2d41-40dd-9c75-5e10397259d6" />


Capture/Record:

* "Application Id"/"Client Id"
* "Secret"

---

## 2. Grant Required Permissions

Your service principal needs:

* Permission to read the dashboard
Share the dashboard with your service account
<img width="879" height="545" alt="image" src="https://github.com/user-attachments/assets/b9aa9fea-a495-4df7-8d8a-7a20a04221f4" />

The service account will need "Can Run" permission


At minimum:
* Grant access to the dashboard you want to publish

Publish the dashboard
* copy the embedded code, from the share panel 
<img width="848" height="154" alt="image" src="https://github.com/user-attachments/assets/27a8d2dc-3e06-49a9-aab9-6d4756568d55" />


---

## 3. Get Dashboard Details

From your Databricks shared dashboard, copy the values :
<img width="873" height="493" alt="image" src="https://github.com/user-attachments/assets/5bc30d90-150f-462c-94f8-803564f45a47" />


* DASHBOARD_ID → from the URL
* WORKSPACE_ID → from workspace settings or URL
* INSTANCE_URL ...etc


---

## 4. Configure GitHub Environment Variables

Go to:

Repo → Settings → Environments

Create an environment called "databicks-service":
Set the Deployment branch to 'main'

Create the following values 

| Name                            | Value               |
| ------------------------------- | ------------------- |
| DATABRICKS_SERVICE_PRINCIPAL_ID | client ID           |


Create the following secretes:

| Name                                | Value         |
| ----------------------------------- | ------------- |
| DATABRICKS_SERVICE_PRINCIPAL_SECRET | client secret |

<img width="789" height="1156" alt="image" src="https://github.com/user-attachments/assets/4ed78441-b899-4c7b-ac37-6cd3bdde6f8a" />


## 4. Configure GitHub Action Variables

Create the following variables 

| Name                            | Value               |
| ------------------------------- | ------------------- |
| DATABRICKS_INSTANCE_URL         | your Databricks URL |
| DATABRICKS_WORKSPACE_ID         | workspace ID        |
| DATABRICKS_DASHBOARD_ID         | dashboard ID        |
| DATABRICKS_EXTERNAL_VIEWER_ID   | github-repo-name    |
| DATABRICKS_EXTERNAL_VALUE       | public-viewer              |

<img width="805" height="761" alt="image" src="https://github.com/user-attachments/assets/e1f54aa4-96df-4965-a94f-b26eb4a7187c" />

---


## 7. How It Works

1. GitHub Actions authenticates with Databricks
2. Retrieves an all-apis token
3. Exchanges it for a scoped dashboard token
4. Injects the token into index.html
5. Deploys the site to GitHub Pages
6. The browser loads the dashboard using that token

---

## Security Notes

* The token is embedded in the final HTML
* It is publicly accessible
* It should be:

  * Short-lived (≤1 hour)
  * Scoped to read-only dashboard access
