# Fediverse to ATProto Crossposter for n8n

This n8n workflow crossposts public original posts from a Mastodon-compatible Fediverse server to an ATProto account.

It works with GoToSocial, Mastodon, and other servers that provide the relevant Mastodon Client API endpoints.

## Features

- polls the source account every five minutes;
- processes only public posts from the configured account;
- excludes replies and boosts/reblogs;
- converts HTML content to plain text while preserving links;
- carries content warnings into the post text;
- splits long posts into ATProto reply threads of no more than 300 grapheme clusters per post;
- creates link and hashtag facets with UTF-8 byte offsets;
- transfers up to four images, including alt text, by uploading every image as a separate ATProto blob before aggregating the results;
- resolves the target PDS dynamically from the ATProto handle and DID document;
- prevents duplicate processing through persistent workflow state and deterministic ATProto record keys;
- retries relevant HTTP requests four times with a five-second delay.

## Limitations

The workflow does not transfer:

- polls;
- video or audio attachments;
- posts with a visibility other than `public`;
- custom emoji image files.

## Source-side behavior

The workflow is deliberately read-only on the Fediverse/ActivityPub side. It performs only authenticated `GET` requests against the Mastodon-compatible account API and downloads explicitly attached images for transfer. All write operations are sent exclusively to the target ATProto PDS.

Only previously unseen, public, original statuses authored by the configured source account are selected:

- existing statuses are recorded as the initial baseline when `bootstrapMode` is `skip_existing`;
- already processed status IDs are ignored on later executions;
- boosts/reblogs are excluded by the API request and rejected again when a returned status contains `reblog` data;
- replies are excluded by the API request and rejected again when `in_reply_to_id` is present;
- likes/favourites are never requested because they are not part of the account-status endpoint;
- follows, bookmarks, notifications, and other account activities are never queried or reproduced.

A native quote post may still be treated as an original authored status. Its text can be crossposted, but the Fediverse quote relationship itself is not transferred.

## Requirements

- a running n8n installation;
- a local account on a Mastodon-compatible Fediverse server;
- a source API user token with read permission;
- an ATProto account and app password;
- the file `fediverse-to-atproto-template.n8n.json`.

The workflow uses only built-in n8n nodes. It has been tested with self-hosted n8n 2.43.x.

## 1. Create the Fediverse API token

### GoToSocial

1. Open `https://YOUR_FEDI_HOST/settings`.
2. Open **Applications** and create a managed application.
3. Use a descriptive name such as `n8n Crossposting`.
4. Grant the application the `read` scope.
5. Add this redirect URI:

   ```text
   https://YOUR_FEDI_HOST/settings/user/applications/callback
   ```

6. Save the application.
7. Open its details and select **Request access token**.
8. Authorize the application and copy the resulting access token.

If **Request access token** is disabled, the settings callback URI is missing. Recreate the application with the exact URI shown above.

### Mastodon

1. Open the preferences of your Mastodon account.
2. Go to **Development** and create a new application.
3. Use a descriptive name such as `n8n Crossposting`.
4. Grant `read`, or the narrower account/status read scopes offered by the server.
5. Save the application and copy its user access token.

Treat the token like a password.

## 2. Create the ATProto app password

1. Sign in to the ATProto account that will receive the crossposts.
2. Open **Settings → Privacy and Security → App Passwords**.
3. Create an app password named, for example, `n8n Crossposting`.
4. Copy the generated app password.

Use the generated app password, not the normal account password. The app password must belong to the same account as `YOUR_ATPROTO_HANDLE`.

## 3. Import the workflow

1. Open n8n.
2. Select **Workflows → Import from File**.
3. Select `fediverse-to-atproto-template.n8n.json`.
4. Keep the workflow inactive until all placeholders and credentials have been configured.

## 4. Fill in the Configuration node

Open the `Configuration` node and replace the three placeholders:

```javascript
const fediBaseUrl = 'https://YOUR_FEDI_HOST';
const fediAccount = 'YOUR_USERNAME@YOUR_FEDI_HOST';
const atprotoHandle = 'YOUR_ATPROTO_HANDLE';
```

Example format:

```javascript
const fediBaseUrl = 'https://social.example';
const fediAccount = 'alice@social.example';
const atprotoHandle = 'alice.example';
```

Rules:

- do not add a trailing slash to `fediBaseUrl`;
- do not add a leading `@` to `fediAccount`;
- do not add a leading `@` to `atprotoHandle`;
- use the full Fediverse account name, including its server domain.

The workflow derives `fediUsername` automatically from `fediAccount`.

The remaining defaults are:

```javascript
bootstrapMode: 'skip_existing',
fetchLimit: 40,
maxProcessedIds: 2000,
language: 'de',
```

Change `language` to the appropriate ISO language code if required.

Do not remove the placeholder validation. It prevents an unfinished template from being activated and then failing in less comprehensible ways several nodes later.

## 5. Create the Fediverse credential in n8n

1. Open the node `Look up Fediverse account`.
2. Under **Credential for Bearer Auth**, select **Create new credential**.
3. Name it:

   ```text
   Fediverse read token
   ```

4. Paste only the source access token into the token field.
5. Do not prefix it with `Bearer`; n8n adds that automatically.
6. Save the credential.
7. Open `Fetch recent Fediverse posts` and select the same credential.

If n8n offers **Allowed HTTP Request Domains**, restrict this credential to the source server domain.

The same credential must be assigned to both source HTTP nodes. Two requests do not require two tokens; bureaucracy has limits, even in automation.

## 6. Create the ATProto credential in n8n

1. Open the node `Create ATProto session`.
2. Under **Credential for Custom Auth**, select **Create new credential**.
3. Name it:

   ```text
   ATProto login body
   ```

4. Paste the following into the credential's **JSON** field:

   ```json
   {
     "body": {
       "identifier": "YOUR_ATPROTO_HANDLE",
       "password": "YOUR_APP_PASSWORD"
     }
   }
   ```

5. Replace `YOUR_ATPROTO_HANDLE` and `YOUR_APP_PASSWORD` with the real values.
6. Keep any hyphens contained in the generated app password.
7. Save the credential and assign it to `Create ATProto session`.

Leave **Allowed HTTP Request Domains** set to `All` for this credential. The workflow resolves the target PDS dynamically, so its hostname is not necessarily known in advance. Restricting the credential to a fixed domain would also break the workflow after a future PDS migration.

The app password is stored in the n8n credential store. It must not be entered directly into the workflow node or exported in the workflow JSON.

## 7. Verify the credential assignments

The final assignment must be:

| Node | Credential |
|---|---|
| `Look up Fediverse account` | `Fediverse read token` |
| `Fetch recent Fediverse posts` | `Fediverse read token` |
| `Create ATProto session` | `ATProto login body` |

Later ATProto nodes use the short-lived session token created during each workflow execution and therefore need no stored credential of their own.

## 8. Save, publish, and establish the baseline

1. Save the workflow.
2. Publish or activate it.
3. Wait for the first scheduled production execution.
4. Confirm that the execution succeeds through at least:

   ```text
   Every 5 minutes
   Configuration
   Look up Fediverse account
   Fetch recent Fediverse posts
   Select new public originals
   ```

The first production execution uses `skip_existing`. It records existing source posts as the duplicate-protection baseline and does not backfill them to ATProto.

Manual test executions do not reliably persist n8n workflow static data. Use a scheduled production execution to establish the baseline.

## 9. Test the crossposter

After the first successful scheduled execution:

1. publish a new public original post on the source server;
2. do not use a reply or boost for the first test;
3. wait for the next five-minute workflow execution;
4. confirm that the ATProto posting nodes were executed;
5. verify the new post on the target account.

Recommended follow-up tests:

- a post containing a URL;
- a post containing hashtags;
- a post with an image and alt text;
- a post containing two to four images with distinct alt texts;
- a post longer than 300 grapheme clusters.

## Troubleshooting

### The execution succeeds, but only the schedule node ran

Open the editor and ensure that the output of `Every 5 minutes` is connected to the input of `Configuration`. Save and publish the corrected workflow.

### `401 AuthenticationRequired: Invalid identifier or password`

The workflow reached the correct ATProto PDS, but the login was rejected. Check that:

- the identifier is the correct ATProto handle;
- the app password was copied completely;
- the app password belongs to that exact ATProto account;
- the normal account password was not used accidentally.

Create a new app password if necessary and update `ATProto login body`.

### No source post is selected

The workflow intentionally ignores:

- posts that are not `public`;
- replies;
- boosts/reblogs;
- posts authored by an account other than `fediAccount`;
- posts already recorded in the duplicate-protection state.

Inspect the output of `Select new public originals` to determine whether the source post passed these filters.

### Existing posts were not copied

This is expected. `bootstrapMode: 'skip_existing'` prevents accidental bulk backfilling during the first production execution.

### Images are missing

Only attachments reported as type `image` are transferred. The workflow accepts no more than four images per source post, matching the legacy ATProto image embed limit. Each image may contain no more than 2,000,000 bytes and must use an `image/*` MIME type.

For correct multi-image processing, verify these two Code-node modes:

| Node | Required mode |
|---|---|
| `Capture uploaded blob` | `Run Once for Each Item` |
| `Build thread items` | `Run Once for All Items` |

The first node preserves one uploaded blob for every input image. The second node sorts and aggregates those blobs into the single `app.bsky.embed.images` object attached to the first ATProto thread post. Using `Run Once for All Items` in both nodes can make n8n lose item correspondence or fail to resolve `$('Prepare image items').item` when multiple images are present.

If `Upload image blob` fails with an HTTP 400 or 413 response, inspect the MIME type and byte size of each individual image. The workflow does not resize or recompress oversized source images.

## Security notes

- Never place OAuth tokens or app passwords directly in Code or HTTP Request nodes.
- Never commit exported n8n credentials to a repository.
- Use a read-only token for the Fediverse source.
- Use an ATProto app password instead of the primary account password.
- Revoke credentials immediately if they are accidentally disclosed.
- Review successful and failed n8n execution data according to your retention requirements.

The template JSON contains placeholder credential references only. It contains no source token, ATProto app password, personal username, handle, or personal domain.

## Relevant documentation

- [n8n HTTP Request credentials](https://docs.n8n.io/integrations/builtin/credentials/httprequest/)
- [Mastodon account API](https://docs.joinmastodon.org/methods/accounts/)
- [Mastodon OAuth](https://docs.joinmastodon.org/spec/oauth/)
- [GoToSocial settings and managed applications](https://docs.gotosocial.org/en/latest/user_guide/settings/)
- [ATProto post creation](https://docs.bsky.app/docs/tutorials/creating-a-post)
