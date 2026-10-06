# Salesforce to ArcGIS with the OAuth Client Credentials flow

*Worked on: August 2026*

I was building a sync that reads constituent records from a Salesforce-based CRM and writes them to ArcGIS Online. It runs unattended, so there's no person around to click "Allow" on a login screen. That ruled out the interactive OAuth flows and pointed me at **Client Credentials**, where the integration authenticates as itself using a client ID and secret.

## The setup

Three pieces, each kept as small as I could make it:

1. **A dedicated, read-only integration user.** Not my admin login and not a shared staff account. If the sync misbehaves or the credentials leak, the blast radius is "someone can read some fields," not "someone can edit the CRM."
2. **A permission set limited to the fields the sync needs.** The default temptation is to give the integration user a broad profile and move on. Instead I granted read access only to the objects and fields the map actually uses. Anything the sync never touches (notes, financial detail, and so on) is invisible to it, so it can't be pulled by accident or by a later bug.
3. **A connected app configured for Client Credentials**, with the integration user set as the "run as" user. The token the sync receives carries exactly that user's permissions, nothing more.

## What was tricky

Most of the friction was in permissions, not code. When a field is missing from the permission set, the API doesn't always shout. A query naming a field the user can't see can fail outright, while a broader describe call just leaves the field out. Both are confusing the first time. My habit now is to check the permission set first whenever a field "doesn't exist."

## Getting a token

The token request is a single POST, which is a nice change from the redirect dance:

```python
import requests

def get_token(login_url, client_id, client_secret):
    resp = requests.post(
        f"{login_url}/services/oauth2/token",
        data={
            "grant_type": "client_credentials",
            "client_id": client_id,
            "client_secret": client_secret,
        },
        timeout=30,
    )
    resp.raise_for_status()
    body = resp.json()
    return body["access_token"], body["instance_url"]
```

Use the returned instance URL for the actual queries rather than hard-coding one.

## Where the credentials live

I chose **AWS Systems Manager Parameter Store** (SecureString parameters) over Secrets Manager. For a single read-only integration with a client ID and secret, Parameter Store does the job: encrypted at rest, access controlled by IAM, and read at runtime instead of baked into code or environment variables. Secrets Manager adds features like managed rotation, which I didn't need for this setup, and it costs more per secret. If rotation becomes a requirement later, moving over is a small change because the code only needs one function that fetches the values.

```python
import boto3

ssm = boto3.client("ssm")

def get_param(name):
    return ssm.get_parameter(Name=name, WithDecryption=True)["Parameter"]["Value"]
```

## Takeaway

For an unattended integration, give it its own identity, make that identity read-only, and trim its permission set to the fields the job needs. Then keep the secret out of the code. Least privilege here is not paperwork; it's what makes a leaked credential a minor incident.
