---
#template: home.html
title: Mastodon Operations
---

# Mastodon Operations

Now that you have your Mastodon instance built, you're going to want to start using it!

## Administrator Access

If you [recall](/building/mastodon/#general), the admin user that we specified in our Mastodon configuration didn't get created when we deployed our instance. Here's how we worked around that:

- Register the account you want to act as the owner of the instance, as a regular user, including confirming your email address.
- Connect to a running `mastodon-web` pod from your CLI machine by entering this:
```bash
~$ kubectl get pods -n mastodon
```
``` {.bash .no-copy}
NAME                                          READY   STATUS
mastodon-media-remove-27895680-w6f95          0/1     Completed
mastodon-media-remove-27905760-2jnnd          0/1     Completed
mastodon-media-remove-27915840-mprc7          0/1     Completed
mastodon-redis-master-0                       1/1     Running
mastodon-redis-replicas-0                     1/1     Running
mastodon-redis-replicas-1                     1/1     Running
mastodon-redis-replicas-2                     1/1     Running
mastodon-sidekiq-all-queues-968db8b7c-77wjg   1/1     Running
mastodon-sidekiq-all-queues-968db8b7c-hh5kz   1/1     Running
mastodon-sidekiq-all-queues-968db8b7c-l7bjc   1/1     Running
mastodon-streaming-bb578b4bc-6pv56            1/1     Running
mastodon-streaming-bb578b4bc-7lc68            1/1     Running
mastodon-streaming-bb578b4bc-swjdh            1/1     Running
mastodon-web-6b7fb7f8d4-2d7gz                 1/1     Running
mastodon-web-6b7fb7f8d4-ds5vm                 1/1     Running
mastodon-web-6b7fb7f8d4-dvr4j                 1/1     Running
```
```bash
~$ kubectl exec -it mastodon-web-6b7fb7f8d4-dvr4j -n mastodon /bin/sh
```
- It doesn't matter which `mastodon-web` pod you connect to, if there is more than one, like in the example above. Once you are connected to a pod, run the following:
```sh
$ RAILS_ENV=production bin/tootctl accounts modify {username} --role Owner
```

This will give your account owner and admin access to your Mastodon instance in the browser. The `Trends`, `Moderation`, `Administration`, `SideKiq`, and `PGHero` menu items are added to the menu in your regular user settings. Get used to keeping an eye on these regularly! A couple of general administrative tips:

- Be sure to walk through the `Server Settings` and set up, at a minumum, your `About` message and your `Server Rules`. Ours are [here](https://mastodon.govsocial.org/about).
- It is **strongly advised** that you add a trusted friend as another instance administrator. Until you do, you are the only person with that access. You will also use the `Roles` menu to manage your growing team of administrators and moderators as your instance grows.
- Keep an eye on your Sidekiq queues, especially anything in `Retries`. There will usually be some sort of error message for those that will help you troubleshoot the issue, particularly if your [SMTP configuration](/building/mastodon/#smtp) isn't working.
- The first time you look at `PGHero`, it will tell you that the `pg-stats` extension needs to be enabled. Do this. Google Cloud SQL installs this extension by default, and if you followed the steps in our [Postgres implementation](/building/mastodon/#postgresql), you should be able to install it right from the browser. This will alert you to any slow-running queries in your database.

## Backups

Upholding the [Mastodon Server Covenant](https://joinmastodon.org/covenant) means ensuring that our instance availability and configuration and our users' data must protected from loss. This means ensuring regular and frequent backups of:

- **Our PostgreSQL database.** We perform automated nightly full database backups using [Cloud SQL backups](https://cloud.google.com/sql/docs/postgres/backup-recovery/backups), with a 7-day retention. We have also enabled [point-in-time recovery](https://cloud.google.com/sql/docs/postgres/backup-recovery/pitr) with 7-day log retention.
- **Our Cloud Storage buckets.** Google Cloud Storage is [geo-redundant](https://cloud.google.com/storage/docs/locations), meaning that data is redundant across at least two zones within at least one geographic place as soon as our users upload it. Our Cloud Storage is used solely for user-uploaded media, such as avatars, banners, and media attached to posts. We believe that geo-redundancy is sufficient to protect this data from loss.
- **GKE Clusters and persistent volume (PV) claims.** We perform automated nightly cluster backups of our instance namespaces using [Backup for GKE](https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/backup-for-gke), with a 7-day retention period.
!!! Warning
    Backup for GKE restores **all** the pods in the the scope of the backup plan, **regardless of status**. If, like we did, you created a lot of terminated and failed pods during your initial deployment, you will want to clear them out to minimize your backup costs. You can get a list of any such pods by running this in from your CLI machine:

    ```bash
    ~$ kubectl get pods --field-selector status.phase=Failed -A
    ```

    You can delete the failed pods by running this from your CLI machine:
    ```bash
    ~$ kubectl delete pods --field-selector status.phase=Failed -A
    ```

## Account Registration Approval

If you leave account registration open on your Mastodon instance this section won't apply, but we wanted to limit accounts on our platform instances to identified and verified public service users and agencies. In order to do this, we require that users sign up with their organizational email address, which we verify against information on the organization's website, among other sources.

We wanted to automate this process as much as we could, partly to make the registration process as seamless as possible for our intended user community, and partly to reduce the administative burden of having to approve every single user.

Many national governments own and administer a top-level or secondary-level DNS domain, use of which already requires that the organization to which a subdomain is being issued meets our criteria for a public service body. These include[^1]:

- [.gov](https://get.gov/)
- [.gov.au](https://www.domainname.gov.au/)
- [.gc.ca](https://en.wikipedia.org/wiki/.gc.ca)
- [.gov.uk](https://www.gov.uk/apply-for-and-manage-a-gov-uk-domain-name)
- [.europa.eu](https://wikis.ec.europa.eu/display/WEBGUIDE/01.+Europa+domain+and+subdomains+overview)

There are also free personal email domains that we are reasonably certain are never used by public service entities. These include[^2]:

- gmail.com
- yahoo.com
- aol.com

We automated the handling of both of these lists with a [Tines](https://www.tines.com/) workflow that uses the [Mastodon API](https://docs.joinmastodon.org/api/) to interact with our instance.

### Tines Automation

In order to automate the processing of registration requests, you will need:

- A [free Tines account](https://login.tines.com/saml_idp/choose_tenant?sign_up=1).
- An OAuth token for an account in your Mastodon instance with the `admin:read:accounts` and `admin:write:accounts` scopes.

#### Mastodon OAuth Token

To generate the OAuth token for your Mastodon account, go to the `Development` item in your Mastodon account settings. Click `New Application`, and enter a suitable account name (eg. `Tines`). When you signed up for Tines and created your tenant, it was given a URL - enter that in the `Application website` field. Leave the `Redirect URI` field at its default setting for now.

Scroll down the list of `Scopes` until you see `admin:read:accounts` and `admin:write:accounts` and check those. You can uncheck everything else in the `Scopes` area.

When you're done, click `Submit`. Edit your newly created application by clicking on it. This will generate and display the OAuth token that you need at the top of the page. The `Client key` and `Client secret` are the values you will need for creating the Tines credential. You don't have to write these down - you'll be able to access them from this menu whenever you need them. Leave this screen open, and switch to your Tines tenant in a different browser tab.

#### Tines Credential

In order for your Tines workflow to authenticate to your Mastodon instance, it will need a credential. You can create one in your Tines team (everyone gets a team in Tines, even if they are an individual user) by expanding your team name and clicking on `Credentials` and then `New Credential` in the screen that appears. Pick `OAuth 2.0` from the dropdown menu.

Give your credential a suitable name, and then get ready to switch back and forth between this screen and your Mastodon `Development` screen as you set this up.

The Tines screen will give you a `Callback URL`. Copy this and paste it into the `Redirect URI` field in your Mastodon screen. While you are in the Mastodon screen, copy and paste both the `Client key` and `Client secret` from there into the `Client ID` and `Client secret` fields respectively in your Tines screen. When you're done, click `Save` in the Mastodon screen, but leave that tab open.

Switch back to Tines, and enter the scopes into the `Scopes` field, exactly like this:

```bash
admin:read:accounts admin:write:accounts
```

Set the remaining values in the Tines screen as follows:

- Set `OAuth Provider` to `Manual`
- Set `Grant type` to `Authorization code`
- Set `OAuth authorization request URL` to `https://{instance-web_domain-value}/oauth/authorize`
- Set `PKCE challenge method` to `SHA-256`
- Set `OAuth token URL` to `https://{instance-web_domain-value}/oauth/token`
- Set `Domains` to `{instance-web_domain-value}`
- Make sure that `All teams & drafts` is checked

When you click `Save`, Tines will generate its OAuth token and, in your Mastodon screen, you should see a request for access which you can accept. Once that is complete, your OAuth token is ready for use in your Tines workflow.

#### Tines Workflow

Workflows in Tines are called "Stories", so click on the `Stories` item in the `Your drafts` part of your Tines menu. To make it easier for you to set everything up, we have exported our Tines workflow as a [JSON](https://www.json.org/json-en.html) file, which you can save locally and import using the `Import` button in this screen[^3].

!!! Note
    You should replace `{my-mastodon-web_domain}`, `{my-credential}`, and `{my-email_address}` with your own values directly in the file **before** importing it. Remember to get rid of the braces `{}` as well!

When you have saved and tested your draft Story, you can publish it in Tines, and you should be all set!

??? Example "Tines Workflow"

    ```json
    {
      "schema_version": 6,
      "standard_lib_version": 13,
      "action_runtime_version": 1,
      "name": "Auto-approve Mastodon accounts",
      "description": "Auto-approves pending Mastodon accounts with email addresses in known domains",
      "guid": "b9700ef606a5f4badfed04ea42155b12",
      "slug": "auto_approve_mastodon_accounts",
      "exported_at": "2023-02-02T21:10:22Z",
      "agents": [
        {
          "type": "Agents::HTTPRequestAgent",
          "name": "Get Pending User Accounts",
          "disabled": false,
          "description": "",
          "guid": "db1336c590099940d47bb2389b634d2b",
          "options": {
            "url": "https://{my-mastodon-web_domain}/api/v2/admin/accounts",
            "content_type": "application_json",
            "method": "get",
            "payload": {
              "origin": "local",
              "status": "pending",
              "limit": 200
            },
            "headers": {
              "Authorization": "Bearer <<CREDENTIAL.{my-credential}>>"
            }
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": [
            {
              "cron": "*/5 * * * *",
              "timezone": "America/Detroit"
            }
          ]
        },
        {
          "type": "Agents::EventTransformationAgent",
          "name": "Extract Pre-Approved Accounts",
          "disabled": false,
          "description": "",
          "guid": "b2f9f20787f3decfe59cc4f5edb041d9",
          "options": {
            "mode": "message_only",
            "loop": false,
            "payload": {
              "pre-approved_accounts": "=FILTER(get_pending_user_accounts.body, LAMBDA(element, MATCH(element.email,\"(.*\\.gov|.*\\.gov\\.au|.*\\.gov\\.uk|.*\\.gc\\.ca)\")))"
            }
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        },
        {
          "type": "Agents::EventTransformationAgent",
          "name": "Explode Pre-Approved Accounts",
          "disabled": false,
          "description": "",
          "guid": "4368d4a7e09e8b0ba1278435502d6c58",
          "options": {
            "mode": "explode",
            "path": "=extract_pre_approved_accounts[\"pre-approved_accounts\"]",
            "to": "individual_item"
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        },
        {
          "type": "Agents::HTTPRequestAgent",
          "name": "Approve Pre-Approved Accounts",
          "disabled": false,
          "description": "",
          "guid": "9e5ad0f2d078d50408a3a91e643c8f40",
          "options": {
            "url": "https://{my-mastodon-web_domain}/api/v1/admin/accounts/<<explode_pre_approved_accounts.individual_item.id>>/approve",
            "content_type": "application_json",
            "method": "post",
            "headers": {
              "Authorization": "Bearer <<CREDENTIAL.{my-credential}>>"
            }
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        },
        {
          "type": "Agents::EventTransformationAgent",
          "name": "Extract Pre-Rejected Accounts",
          "disabled": false,
          "description": "",
          "guid": "fcfe50b2803bc32077c04dce841a33f2",
          "options": {
            "mode": "message_only",
            "loop": false,
            "payload": {
              "pre-rejected_accounts": "=FILTER(get_pending_user_accounts.body, LAMBDA(element, MATCH(element.email,\"(.*\\gmail\\.com|.*\\aol\\.com|.*\\yahoo\\.com)\")))"
            }
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        },
        {
          "type": "Agents::EventTransformationAgent",
          "name": "Explode Pre-Rejected Accounts",
          "disabled": false,
          "description": "",
          "guid": "5c672894c3e6af04e43a7e5b69de1bd9",
          "options": {
            "mode": "explode",
            "path": "=extract_pre_rejected_accounts[\"pre-rejected_accounts\"]",
            "to": "individual_item"
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        },
        {
          "type": "Agents::HTTPRequestAgent",
          "name": "Reject Pre-Rejected Accounts",
          "disabled": false,
          "description": "",
          "guid": "c1958d276321136a1a86833ec33a87ea",
          "options": {
            "url": "https://{my-mastodon-web_domain}/api/v1/admin/accounts/<<explode_pre_rejected_accounts.individual_item.id>>/reject",
            "content_type": "application_json",
            "method": "post",
            "headers": {
              "Authorization": "Bearer <<CREDENTIAL.{my-credential}>>"
            }
          },
          "reporting": {
            "time_saved_value": 0,
            "time_saved_unit": "minutes"
          },
          "monitoring": {
            "monitor_all_events": false,
            "monitor_failures": false,
            "monitor_no_events_emitted": null
          },
          "width": null,
          "schedule": null
        }
      ],
      "diagram_notes": [],
      "links": [
        {
          "source": 0,
          "receiver": 1
        },
        {
          "source": 0,
          "receiver": 4
        },
        {
          "source": 1,
          "receiver": 2
        },
        {
          "source": 2,
          "receiver": 3
        },
        {
          "source": 4,
          "receiver": 5
        },
        {
          "source": 5,
          "receiver": 6
        }
      ],
      "diagram_layout": "{\"db1336c590099940d47bb2389b634d2b\":[495,45],\"b2f9f20787f3decfe59cc4f5edb041d9\":[330,135],\"4368d4a7e09e8b0ba1278435502d6c58\":[330,225],\"9e5ad0f2d078d50408a3a91e643c8f40\":[330,330],\"fcfe50b2803bc32077c04dce841a33f2\":[585,135],\"5c672894c3e6af04e43a7e5b69de1bd9\":[585,225],\"c1958d276321136a1a86833ec33a87ea\":[585,330]}",
      "send_to_story_enabled": false,
      "entry_agent_guid": null,
      "exit_agent_guids": [],
      "exit_agent_guid": null,
      "keep_events_for": 604800,
      "reporting_status": true,
      "send_to_story_access": null,
      "story_library_metadata": {},
      "recipients": [
        "{my-email_address}"
      ],
      "story_level_monitoring_enabled": false,
      "monitor_failures": false,
      "send_to_stories": [],
      "form": null,
      "forms": []
    }
    ```

## Server Moderation

As part of creating an instance safe for public service users and agencies using their public personas, it is important that we can maintain a reliable server moderation policy, limiting federation with instances that are conflict with [our server rules](https://mastodon.govsocial.org/about/).

Also, as a small team, we wanted to automate the maintainance of our base server moderation list from these sources, to allow us to dedicate our resources to the moderation of our specific instance. Enter the [FediBlockHole](https://github.com/eigenmagic/fediblockhole) project[^4].

### FediBlockHole

We have documented how we containerized and deployed FediBlockHole in our cluster [here](/building/fediblockhole/). Once that was done, we needed to decide which lists to pull from.

#### Selecting Block Lists

Server moderation in the Fediverse is a little like spam filtering for email servers. It is generally up to each individual mailop to decide and implement their own spam filtering policy, with widely varying results. Over time, trusted lists of "known bad" servers (such as [these](https://mxtoolbox.com/problem/blacklist/)) have evolved to provide a more consistent and centralized way of filtering traffic from poorly-behaved MTAs.

[The RapidBlock Project](https://rapidblock.org/) is an equivalent of this for Mastodon instances, and was a clear choice for us. You can see where we pull this list in our [FediBlockHole configuration file](/building/fediblockhole/#deploying-fediblockhole) here:

```toml
blocklist_url_sources = [
  # { url = 'file:///path/to/fediblockhole/samples/demo-blocklist-01.csv', format = 'csv' },
  { url = 'https://rapidblock.org/blocklist.json', format = 'rapidblock.json' },
]
```

The `rapidblock.json` format is specific to RapidBlock (it is designed for use by Fediverse platforms as a whole, and not just Mastodon), and looks like this:

``` {.json .no-copy}
{
  "@spec": "https://rapidblock.org/spec/v1/",
  "publishedAt": "2022-12-29T18:40:02.065805293Z",
  "blocks": {
    "101010.pl": {
      "isBlocked": true,
      "reason": "cryptomining javascript, white supremacy",
      "tags": [
        "antisemitism",
        "malware",
        "racism"
      ],
      "dateRequested": "2022-11-16T00:00:00Z",
      "dateDecided": "2022-11-16T00:00:00Z"
    ...
```

!!! Note
    Mastodon server moderation has severity levels that are explained [here](https://docs.joinmastodon.org/admin/moderation/#server-wide-moderation), but you will notice from the sample RapidBlock file that no severity level is specified. This means that FediBlockHole will use the `max_severity` configuration setting, which is `suspend` by default and, if users on your instance are following accounts from the moderated domain, the `max_followed_severity` setting which defaults to `silence`.

We also came across a set of consolidated blocklists created (using FediBlockHole) from the server block lists from various "tiers" of Mastodon instances. [The Oliphant Social Blocklist](https://writer.oliphant.social/oliphant/the-oliphant-social-blocklist) provides a number of different lists using different tiers and [merge plans](#selecting-a-merge-plan) for each.

And here is where the fun started. Being new, and sensitive to our goal of creating an instance safe for public service users and agencies using their public personas, we initially selected the [Unified Max Block List](https://codeberg.org/oliphant/blocklists/raw/branch/main/blocklists/_unified_max_blocklist.csv). This is all blocks from all tiers, using a `max` merge plan.

This resulted in a huge blocklist, including the instance (mstdn.ca) where at least one of our team has their personal account! As noted [here](/building/fediblockhole/#deploying-fediblockhole), FediBlockHole has no mechanism for removing blocks that are no longer in the subscribed lists, so we had to figure out what to do next.

Luckily, there is a [collection of Python-based utilities](https://github.com/ineffyble/mastodon-block-tools), from which we got a [script](https://github.com/chdorner/secretbearsociety/blob/main/clear-blocks.py) that, once we added a timeout to avoid tripping the rate-limit on our API, did the trick.

We now use the much more limited but still effective [Unified Min Blocklist](https://codeberg.org/oliphant/blocklists/raw/branch/main/blocklists/_unified_max_blocklist.csv), which combines block lists from tiers 0 through 2 servers with a `min` merge plan. You can see where we pull this list in our [FediBlockHole configuration file](/building/fediblockhole/#deploying-fediblockhole) here:

```toml
blocklist_url_sources = [
  # { url = 'file:///path/to/fediblockhole/samples/demo-blocklist-01.csv', format = 'csv' },
            { url = 'https://codeberg.org/oliphant/blocklists/raw/branch/main/blocklists/_unified_min_blocklist.csv', format = 'csv' },
]
``` 

!!! Note
    This list does **NOT** include the RapidBlock list, so we combine it with the above list, using a `max` merge plan

#### Selecting a Merge Plan

One of the great features of FediBlockHole is the ability to specify a `mergeplan` setting of `min` or `max` when combining lists from multiple sources. The default is `max`.

The difference between the two is how FediBlockHole resolves the conflict when the same block is encountered from different sources with different [severities](https://docs.joinmastodon.org/admin/moderation/#server-wide-moderation). A `max` merge plan selects the **highest** of the severity levels from all sources of the same block, while a `min` merge plan selects the **lowest**.

We have opted to implement a `max` merge plan for our lists.

[^1]: We are keen to add more - please [let us know](mailto:cunningpike@gmail.com) if there are any that should be added. Our worldview is regrettably US-centric, and there are likely many others that we have missed.
[^2]: This list is subject to change without notice, but we will keep our documentation of it, both here and on our instances, up to date.
[^3]: We haven't tried this ourselves so, if you do, [let us know](mailto:cunningpike@gmail.com) how it went!
[^4]: Hat tip to [oliphant@oliphant.social](https://oliphant.social/@oliphant) for pointing us at this!