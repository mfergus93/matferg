# Visitor IP logger

Cloudflare Worker that runs in front of `matferg.com`, records HTML visits in
Cloudflare D1, and forwards each request to the existing GitHub Pages origin.

Only top-level HTML page visits are recorded. Each record includes the
timestamp, IP address, approximate Cloudflare location, path (without its query
string), sanitized referrer, user agent, network, and classification evidence.
High-confidence scanner requests are blocked and written to a separate security
event table. Records are deleted after 30 days by default. The log-reading and
email-test endpoints require a secret bearer token.

## Before deployment

1. Create a Cloudflare account and add `matferg.com` as a zone.
2. In Cloudflare DNS, create a proxied CNAME from `@` to
   `mfergus93.github.io`. Keep the repository's `CNAME` file unchanged.
3. Install Node.js if `npx` is not already available.
4. From this directory, authenticate using Cloudflare's browser flow:

   ```powershell
   npx wrangler@latest login
   ```

Never paste your Cloudflare password, global API key, or API token into chat or
commit one to the repository.

## Create and configure D1

Run:

```powershell
npx wrangler@latest d1 create matferg-visitors
```

Copy the returned database ID into `wrangler.jsonc`, replacing
`REPLACE_WITH_DATABASE_ID`, and initialize the remote database:

```powershell
npx wrangler@latest d1 execute matferg-visitors --remote --file schema.sql
```

For an existing database created from the older schema, apply the tracked
migration once instead:

```powershell
npx wrangler@latest d1 execute matferg-visitors --remote --file migrations/0001_harden_logging.sql
```

Create the private log-access token. Wrangler prompts for the value without
putting it in the config file:

```powershell
npx wrangler@latest secret put LOG_API_TOKEN
```

Use a randomly generated value of at least 32 bytes and store it in a password
manager.

## Deploy

```powershell
npx wrangler@latest deploy
```

The route in `wrangler.jsonc` requires the `matferg.com` DNS record to be
proxied through Cloudflare.

## Read logs

PowerShell example:

```powershell
.\view-logs.ps1 -Limit 100
```

Filter by exact IP with `./view-logs.ps1 -IpAddress 203.0.113.10 -Limit 100`.
The API returns at most 500 records per request and sends
`Cache-Control: no-store`. The helper reads a Windows user-encrypted local
token file that is excluded from Git.

## Privacy and security

Raw IP addresses may constitute personal data. The site links to an accurate
privacy notice. Keep retention short, do not publish the token or database,
restrict Cloudflare account access, and use the data only for the disclosed
security and audience-measurement purposes. Classification is probabilistic and
does not establish a visitor's real-world identity.
