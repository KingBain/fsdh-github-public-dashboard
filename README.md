# Databricks Dashboard Integration Setup

This project dynamically injects a short-lived Databricks token into a static site during the GitHub Actions deployment.

To make this work, you need:

1. A Databricks Service Principal
2. A published Databricks dashboard
3. GitHub repository variables + secrets configured

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
<img width="856" height="76" alt="image" src="https://github.com/user-attachments/assets/8fb7b0cb-b499-4965-8fec-ce285da90bda" />

The service account will need "Can Run" permission


At minimum:
* Grant access to the dashboard you want to publish

Publish the dashboard
* copy the embedded code, from the share panel 
<img width="868" height="528" alt="image" src="https://github.com/user-attachments/assets/4555c59f-b0fb-4001-b489-32cbca68f5b5" />


---

## 3. Get Dashboard Details

From your Databricks shared dashboard, copy the values :
<img width="873" height="493" alt="image" src="https://github.com/user-attachments/assets/5bc30d90-150f-462c-94f8-803564f45a47" />


* DASHBOARD_ID → from the URL
* WORKSPACE_ID → from workspace settings or URL
* INSTANCE_URL → e.g.

```
https://adb-xxxxxxxx.x.azuredatabricks.net
```

---

## 4. Configure GitHub Environment Variables

Go to:

Repo → Settings → Environments

Create an environment called "github-pages"(if it doesnt already exist:
Create the following values 

| Name                            | Value               |
| ------------------------------- | ------------------- |
| DATABRICKS_INSTANCE_URL         | your Databricks URL |
| DATABRICKS_WORKSPACE_ID         | workspace ID        |
| DATABRICKS_DASHBOARD_ID         | dashboard ID        |
| DATABRICKS_SERVICE_PRINCIPAL_ID | client ID           |
| DATABRICKS_EXTERNAL_VIEWER_ID   | public-user         |
| DATABRICKS_EXTERNAL_VALUE       | viewer              |


Create the following secretes:

| Name                                | Value         |
| ----------------------------------- | ------------- |
| DATABRICKS_SERVICE_PRINCIPAL_SECRET | client secret |


<img width="458" height="806" alt="image" src="https://github.com/user-attachments/assets/8461c2f9-7107-48e5-aa28-edea3f893c8f" />


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

---

# GitHub Actions Workflow (Environment-Based)

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    permissions:
      contents: read
      pages: write
      id-token: write

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Generate Databricks scoped access token
        id: databricks_token
        shell: bash
        env:
          INSTANCE_URL: ${{ vars.DATABRICKS_INSTANCE_URL }}
          WORKSPACE_ID: ${{ vars.DATABRICKS_WORKSPACE_ID }}
          DASHBOARD_ID: ${{ vars.DATABRICKS_DASHBOARD_ID }}
          SERVICE_PRINCIPAL_ID: ${{ vars.DATABRICKS_SERVICE_PRINCIPAL_ID }}
          SERVICE_PRINCIPAL_SECRET: ${{ secrets.DATABRICKS_SERVICE_PRINCIPAL_SECRET }}
          EXTERNAL_VIEWER_ID: ${{ vars.DATABRICKS_EXTERNAL_VIEWER_ID }}
          EXTERNAL_VALUE: ${{ vars.DATABRICKS_EXTERNAL_VALUE }}
        run: |
          set -euo pipefail

          INSTANCE_URL="${INSTANCE_URL%/}"

          basic_auth="$(printf '%s:%s' "$SERVICE_PRINCIPAL_ID" "$SERVICE_PRINCIPAL_SECRET" | base64 | tr -d '\n')"

          echo "Requesting Databricks all-apis token..."
          oidc_response="$(
            curl -sS \
              -X POST "${INSTANCE_URL}/oidc/v1/token" \
              -H "Authorization: Basic ${basic_auth}" \
              -H "Content-Type: application/x-www-form-urlencoded" \
              --data-urlencode "grant_type=client_credentials" \
              --data-urlencode "scope=all-apis"
          )"

          oidc_token="$(echo "$oidc_response" | jq -r '.access_token')"

          if [ -z "$oidc_token" ] || [ "$oidc_token" = "null" ]; then
            echo "Failed to get Databricks all-apis token"
            echo "$oidc_response"
            exit 1
          fi

          echo "::add-mask::$oidc_token"

          echo "Requesting dashboard tokeninfo..."
          tokeninfo_response="$(
            curl -sS -G \
              "${INSTANCE_URL}/api/2.0/lakeview/dashboards/${DASHBOARD_ID}/published/tokeninfo" \
              -H "Authorization: Bearer ${oidc_token}" \
              --data-urlencode "external_viewer_id=${EXTERNAL_VIEWER_ID}" \
              --data-urlencode "external_value=${EXTERNAL_VALUE}"
          )"

          authorization_details="$(echo "$tokeninfo_response" | jq -c '.authorization_details')"

          params_json="$(echo "$tokeninfo_response" | jq -c 'del(.authorization_details) | with_entries(select(.value != null))')"

          form_body="$(
            jq -rn \
              --argjson params "$params_json" \
              --arg authorization_details "$authorization_details" '
                ($params + {
                  grant_type: "client_credentials",
                  authorization_details: $authorization_details
                })
                | to_entries
                | map(
                    .key as $k
                    | if (.value | type) == "array" then
                        (.value | map("\($k|@uri)=\(.|tostring|@uri)") | join("&"))
                      else
                        "\($k|@uri)=\(.value|tostring|@uri)"
                      end
                  )
                | join("&")
              '
          )"

          scoped_response="$(
            curl -sS \
              -X POST "${INSTANCE_URL}/oidc/v1/token" \
              -H "Authorization: Basic ${basic_auth}" \
              -H "Content-Type: application/x-www-form-urlencoded" \
              --data "$form_body"
          )"

          scoped_token="$(echo "$scoped_response" | jq -r '.access_token')"

          if [ -z "$scoped_token" ] || [ "$scoped_token" = "null" ]; then
            echo "Failed to get scoped token"
            echo "$scoped_response"
            exit 1
          fi

          echo "::add-mask::$scoped_token"

          {
            echo "DATABRICKS_SCOPED_TOKEN=$scoped_token"
            echo "DATABRICKS_INSTANCE_URL=$INSTANCE_URL"
            echo "DATABRICKS_WORKSPACE_ID=$WORKSPACE_ID"
            echo "DATABRICKS_DASHBOARD_ID=$DASHBOARD_ID"
          } >> "$GITHUB_ENV"

      - name: Inject token into HTML
        run: |
          set -euo pipefail

          escape() { printf '%s' "$1" | sed 's/[\/&]/\\&/g'; }

          sed -i.bak \
            -e "s/__DATABRICKS_INSTANCE_URL__/$(escape "$DATABRICKS_INSTANCE_URL")/g" \
            -e "s/__DATABRICKS_WORKSPACE_ID__/$(escape "$DATABRICKS_WORKSPACE_ID")/g" \
            -e "s/__DATABRICKS_DASHBOARD_ID__/$(escape "$DATABRICKS_DASHBOARD_ID")/g" \
            -e "s/__DATABRICKS_TOKEN__/$(escape "$DATABRICKS_SCOPED_TOKEN")/g" \
            index.html

          rm -f index.html.bak

      - name: Setup Pages
        uses: actions/configure-pages@v6

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: "."

      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v5
```
