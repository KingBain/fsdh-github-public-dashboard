# Databricks Dashboard Integration Setup

This project dynamically injects a short-lived Databricks token into a static site during the GitHub Actions deployment.

To make this work, you need:

1. A Databricks Service Principal
2. A published Databricks dashboard
3. GitHub repository variables + secrets configured

---

## 1. Create a Databricks Service Principal

1. Go to your Databricks workspace
2. Navigate to:

   * **Settings → Identity and Access → Service Principals**
3. Create a new service principal

Capture:

* Application (client) ID
* Client secret

---

## 2. Grant Required Permissions

Your service principal needs:

* Access to the workspace
* Permission to read the dashboard
* Permission to generate tokens (all-apis scope)

At minimum:

* Add it to the workspace
* Grant access to the dashboard you want to publish

---

## 3. Get Dashboard Details

From your Databricks dashboard:

* DASHBOARD_ID → from the URL
* WORKSPACE_ID → from workspace settings or URL
* INSTANCE_URL → e.g.

```
https://adb-xxxxxxxx.x.azuredatabricks.net
```

---

## 4. Configure GitHub Variables

Go to:

Repo → Settings → Variables → Actions

Create:

| Name                            | Value               |
| ------------------------------- | ------------------- |
| DATABRICKS_INSTANCE_URL         | your Databricks URL |
| DATABRICKS_WORKSPACE_ID         | workspace ID        |
| DATABRICKS_DASHBOARD_ID         | dashboard ID        |
| DATABRICKS_SERVICE_PRINCIPAL_ID | client ID           |
| DATABRICKS_EXTERNAL_VIEWER_ID   | public-user         |
| DATABRICKS_EXTERNAL_VALUE       | viewer              |

---

## 5. Configure GitHub Secrets

Go to:

Repo → Settings → Secrets → Actions

Create:

| Name                                | Value         |
| ----------------------------------- | ------------- |
| DATABRICKS_SERVICE_PRINCIPAL_SECRET | client secret |

---

## 6. Configure GitHub Environment (Recommended)

1. Go to:

   * Settings → Environments
2. Create:

```
github-pages
```

3. Add:

* Variables
* Secrets

4. Optional:

* Add required reviewers for deployment approval

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
