# Databricks Dashboard Integration Proof of Concept

This project dynamically injects a short-lived Databricks OAuth token into a static HTML site during GitHub Actions deployment. This allows you to publish a Databricks dashboard to GitHub Pages for public consumption.

### Important Considerations
https://learn.microsoft.com/en-us/azure/databricks/dashboards/share/embedding/external-embed
Based on the official documentation and this GitHub Actions implementation, here are the critical bullets a user should be aware of when using this Proof of Concept:

*   **Compute Costs & Billing Ownership**: Although hosting on GitHub Pages is free, the dashboard itself is **not**. Every time a user views the dashboard, it executes queries on your **Databricks SQL Warehouse**. The organization that owns the Databricks workspace is responsible for the DBU (Databricks Unit) costs associated with those queries, regardless of who is viewing the GitHub site.
*   **Token Expiration & Static Limits**: Because GitHub Pages is a static host, the Databricks token is semi "frozen" into the HTML at the time of deployment. Since these user-scoped tokens are designed to be short-lived (typically 1 hour), the dashboard will stop working if a user leaves the page open for too long. This project relies on the **GitHub Action Schedule** to "refresh" the page with a fresh token twice an hour.
*   **Compute vs. Data Permissions**: This model uses a split-permission logic. The **Compute** (the SQL Warehouse) always runs using the credentials of the person who **published** the dashboard. However, the **Data Access** (the tables) is checked against the **Service Principal**. You must ensure your Service Principal has  privileges on the underlying dashboard or the charts will return an "Access Denied" error.
*   **Compute Source & Auto-Stop**: All data processing happens on **Databricks SQL Warehouses**, not on the end-user's computer or GitHub's servers. The user's browser only receives the final visual results. To avoid unexpected costs when no one is visiting your site, ensure your SQL Warehouse is configured to **"Auto-Stop"** after a period of inactivity.
*   **Concurrency & Rate Limits**: Databricks imposes a hard limit of **20 simultaneous dashboard loads per second** for external embedding. While this is sufficient for most use cases, if the site receives a massive spike in traffic, users may see "Rate Limit Exceeded" errors because the single Service Principal identity is being used to handle all requests.

### Prerequisites
To make this work, you need:
1. A Databricks Service Principal.
2. A published Databricks dashboard.
3. A GitHub repository with Environments, Variables, and Secrets configured.

---

## Demo Site
<details>
<summary><b>Click to view Demo Site Screenshot</b></summary>
<br>
<img width="1188" alt="Demo Site" src="https://github.com/user-attachments/assets/cc3d2738-464c-402e-b7f6-a39fca3d281d" />
</details>

---

## Step 1: Create a Databricks Service Principal

1. Go to your Databricks workspace.
2. Navigate to **Settings → Workspace admin → Identity and Access → Service Principals**.
3. Create a new service principal. Ensure you select **none** of the entitlements.
4. Generate an OAuth "Secret" Private Key.

<details>
<summary><b>Click to view walkthrough screenshots</b></summary>
<br>
<img width="863" alt="Workspace Menu" src="https://github.com/user-attachments/assets/85a6479b-71fa-4b8b-b94f-b097b88ba2cc" />
<img width="1018" alt="Create Service Principal" src="https://github.com/user-attachments/assets/9972174e-ce59-434a-9f8f-922a9efeb556" />
<img width="986" alt="OAuth Secret 1" src="https://github.com/user-attachments/assets/4b0a6019-48a5-47e8-b583-73a0a859371a" />
<img width="1061" alt="OAuth Secret 2" src="https://github.com/user-attachments/assets/9908e121-2d41-40dd-9c75-5e10397259d6" />
</details>

** Important:** Securely capture and save the following values:
* `Application Id` (Client ID)
* `Secret`

---

## Step 2: Grant Required Permissions

Your newly created service principal needs permission to read the dashboard. 

1. Navigate to the Databricks dashboard you want to publish.
2. Open the **Share** panel.
3. Grant your service principal **Can Run** permissions.
4. **Publish** the dashboard.
5. Copy the embedded code from the share panel for the next step.

<details>
<summary><b>Click to view Sharing screenshots</b></summary>
<br>
<img width="879" alt="Share Dashboard" src="https://github.com/user-attachments/assets/b9aa9fea-a495-4df7-8d8a-7a20a04221f4" />
<img width="848" alt="Embedded Code" src="https://github.com/user-attachments/assets/27a8d2dc-3e06-49a9-aab9-6d4756568d55" />
</details>

---

## Step 3: Get Dashboard Details

From your Databricks shared dashboard URL and settings, locate and copy these values:

* `DASHBOARD_ID` → Found in the dashboard URL.
* `WORKSPACE_ID` → Found in workspace settings or the URL.
* `INSTANCE_URL` → Your base Databricks URL (e.g., `https://adb-123456789.9.azuredatabricks.net`).

<details>
<summary><b>Click to view URL breakdown</b></summary>
<br>
<img width="873" alt="Dashboard Details" src="https://github.com/user-attachments/assets/5bc30d90-150f-462c-94f8-803564f45a47" />
</details>

---

## Step 4: Configure GitHub Environment Secrets

1. In your GitHub repository, go to **Settings → Environments**.
2. Create a new environment called `databricks-dashboard`.
3. Set the **Deployment branch** rule to `main`.
4. Add the following **Environment Variables / Secrets**:

### Environment Variable
| Name | Value |
|------|-------|
| `DATABRICKS_SERVICE_PRINCIPAL_ID` | *Your Client ID* |

### Environment Secret
| Name | Value |
|------|-------|
| `DATABRICKS_SERVICE_PRINCIPAL_SECRET` | *Your Client Secret* |

<details>
<summary><b>Click to view Environment Setup</b></summary>
<br>
<img width="792" height="1167" alt="image" src="https://github.com/user-attachments/assets/e6f53f39-bb3a-4249-9782-5a19fa9631b1" />
</details>

---

## Step 5: Configure GitHub Repository Variables

Go to **Settings → Secrets and variables → Actions** and select the **Variables** tab. Create the following Repository Variables:

| Name | Value |
|------|-------|
| `DATABRICKS_INSTANCE_URL` | *Your Databricks Instance URL* |
| `DATABRICKS_WORKSPACE_ID` | *Your Workspace ID* |
| `DATABRICKS_DASHBOARD_ID` | *Your Dashboard ID* |
| `DATABRICKS_EXTERNAL_VIEWER_ID` | `github-repo-name` (or a custom identifier) |
| `DATABRICKS_EXTERNAL_VALUE` | `public-viewer` |

<details>
<summary><b>Click to view Action Variables</b></summary>
<br>
<img width="796" height="736" alt="image" src="https://github.com/user-attachments/assets/fc48bcbf-b532-4a64-87ff-9bb5fdbd8ee0" />
</details>

---

## How It Works

1. **Authentication**: GitHub Actions securely authenticates with Databricks using basic auth (Base64 URL-encoded).
2. **Token Fetching**: It retrieves a broad `all-apis` token.
3. **Token Exchange**: It immediately exchanges that token for a strictly scoped dashboard viewer token using `tokeninfo`.
4. **Injection**: A `sed` script finds placeholders in `index.html` and injects the scoped token.
5. **Deployment**: The static site is deployed to GitHub Pages.
6. **Viewing**: The user's browser automatically loads the Databricks dashboard using the injected token.

---

## Security Notes

* **Public Exposure**: The generated token is embedded directly in the final HTML and is publicly accessible.
* **Short-Lived**: The GitHub action is scheduled via cron to rebuild twice an hour. Databricks tokens are short-lived (usually ≤ 1 hour) limiting the window of risk.
* **Strictly Scoped**: The token generated by this pipeline is *only* capable of viewing the specific published dashboard. It cannot access underlying data, notebooks, or APIs.
* **Note**: This guide was written with the help of an LLM.
