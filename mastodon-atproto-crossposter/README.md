# Mastodon → ATProto Crossposter for n8n

An importable workflow for copying public original Mastodon posts to a configurable ATProto account. Contains no personal account settings or secrets. Inactive on import.

## Features

- Polls every two minutes; public originals only, excluding replies and boosts.
- Converts HTML to text, preserving links and including content warnings.
- Splits threads at grapheme boundaries, with at most 300 graphemes per part including numbering, using root and parent references.
- Creates link and hashtag facets with UTF-8 byte offsets.
- Transfers up to four images with alt text, compressing to JPEG and checking each image is below 1 MB.
- Resolves handle, DID, and PDS dynamically and verifies the handle in the DID document.
- Uses persistent processing state, deterministic record keys, HTTP timeouts, and bounded retries.

## Requirements

Self-hosted n8n with built-in HTTP Request, Code, and Edit Image nodes, plus GraphicsMagick. The original template was previously tested with n8n 2.43.x. This generic edition has been checked for JSON structure, code syntax, and connections; a separate live import has not been tested.

## 1. Import and configure

Import `mastodon-to-atproto.n8n.json` using **Import from File**. In **Configuration**, replace:

```javascript
const fediBaseUrl = 'https://YOUR_MASTODON_HOST';
const fediAccount = 'YOUR_USERNAME@YOUR_MASTODON_HOST';
const atprotoHandle = 'YOUR_ATPROTO_HANDLE';
```

Include HTTPS in the base URL, and omit a leading @ from the account and handle. Example: `https://social.example`, `alice@social.example`, `alice.example`. Never store secrets here. Placeholder validation prevents unconfigured runs.

The default post language is `de`; change `language` to your content language, such as `en`. This is post metadata, not the interface language. The scheduling timezone defaults to `Europe/Berlin` and can be changed in workflow settings.

## 2. Mastodon credential

Under **Preferences → Development → New application**, create an application with only `read:accounts` and `read:statuses`. Remove other default permissions and keep the default redirect URI. Save and copy the user access token.

Create an n8n **Bearer Auth** credential. Paste the token without the word Bearer. If supported, restrict allowed domains to your Mastodon host. Assign the same credential to:

- **Look up Fediverse account**
- **Fetch recent Fediverse posts**

Client ID and client secret are not required. Mastodon requests are read-only.

## 3. ATProto credential

Create a dedicated app password with your ATProto provider. In **Create ATProto session**, create a **Custom Auth** credential and enter this in its JSON field:

```json
{
  "body": {
    "identifier": "YOUR_ATPROTO_HANDLE",
    "password": "YOUR_APP_PASSWORD"
  }
}
```

Replace both placeholders. The handle must match Configuration. Preserve password hyphens. Assign the credential to this node; subsequent nodes automatically use its temporary session token.

The destination is the PDS resolved from the DID document, which may differ from the handle domain. Inspect **Resolve ATProto handle → Build DID document URL → Fetch DID document → Extract PDS endpoint** to verify the PDS. Restrict the credential to that verified PDS host if supported. Recheck after a PDS migration. Never send the password to an unverified host.

## 4. Activate and test

1. Disable any previous crossposter for this source.
2. Save and publish or activate the workflow.
3. Wait for the first scheduled production run. With `bootstrapMode: 'skip_existing'`, fetched existing posts become the baseline and are not published.
4. Publish a new public original post and check the target after the next successful poll.
5. Test a long post and four images with distinct alt texts. Replies, boosts, and non-public posts must not appear.

Manual runs do not reliably persist static data. Establish the baseline with a scheduled production run. Prevent overlapping executions and run only one crossposter for the same source and target.

## Recovery and limitations

Completed source IDs are stored in workflow static data. Deterministic keys `masto-SOURCE_ID-PART_INDEX` mean repeated putRecord requests write the same record. After a partial failure, a later run reconstructs the thread under the same keys. Do not change source text or splitting logic during recovery. Later edits and deletions are not synchronized. Concurrent workflow copies are unsupported.

Each poll fetches at most 40 recent statuses without full backward pagination. Extended downtime or more than 40 new statuses between successful polls can cause missed posts. A fresh import has its own state and establishes a new baseline.

Exhausted HTTP retries stop execution. The next scheduled run attempts unfinished posts again. Automatic Retry-After handling and dedicated error notifications are not included. Inspect **Executions** and optionally configure an n8n error workflow.

JPEG conversion removes animation and transparency. Images still exceeding the limit cause an error; reduce `imageMaxDimension` or `imageJpegQuality` in Configuration. Video, audio, polls, and native quote relationships are not transferred. HTML conversion targets typical Mastodon post markup.

## Sharing and secrets

The supplied JSON and README are generic templates suitable for sharing. Each recipient creates their own credentials. Before sharing a configured export, inspect it for personal values, pinned data, and credential references.

Successful and manual executions are not saved by default. Failed execution data may contain temporary session tokens; restrict access and retention.

## Documentation

- [Mastodon Accounts API](https://docs.joinmastodon.org/methods/accounts/)
- [ATProto posts and threads](https://docs.bsky.app/docs/tutorials/creating-a-post)
- [n8n HTTP Request credentials](https://docs.n8n.io/integrations/builtin/credentials/httprequest/)
- [n8n Edit Image](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage/)
