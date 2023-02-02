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

It doesn't matter which `mastodon-web` pod you connect to, if there is more than one, like in the example above. Once you are connected to a pod, run the following:

```sh
$ RAILS_ENV=production bin/tootctl accounts modify {username} --role Owner
```

This will give your account owner and admin access to your Mastodon instance in the browser. The `Trends`, `Moderation`, `Administration`, `SideKiq`, and `PGHero` menu items are added to the menu in your regular user settings. Get used to keeping an eye on these regularly! A couple of general administrative tips:

- Be sure to walk through the `Server Settings` and set up, at a minumum, your `About` message and your `Server Rules`. Ours are [here](https://mastodon.govsocial.org/about).
- It is **strongly advised** that you add a trusted friend as another instance administrator. Until you do, you are the only person with that access. You will also use the `Roles` menu to manage your growing team of administrators and moderators as your instance grows.
- Keep an eye on your Sidekiq queues, especially anything in `Retries`. There will usually be some sort of error message for those that will help you troubleshoot the issue, particularly if your [SMTP configuration](/building/mastodon/#smtp) isn't working.
- The first time you look at `PGHero`, it will tell you that the `pg-stats` extension needs to be enabled. Do this. Google Cloud SQL installs this extension by default, and if you followed the steps in our [Postgres implementation](/building/mastodon/#postgresql), you should be able to install it right from the browser. This will alert you to any slow-running queries in your database.

## Account Registration Approval

If you leave account registration open on your Mastodon instance this section won't apply, but we wanted to limit accounts on our platform instances to identified and verified public service users and agencies. In order to do this, we require that users sign up with their organizational email address, which we verify against information on the organization's website, among other sources.

We wanted to automate this process as much as we could, partly to make the registration process as seamless as possible for our intended user community, and partly to reduce the administative burden of having to approve every single user.

Many national governments own and administer a top-level or secondary-level DNS domain, use of which already requires that the organization to which a subdomain is being issued meets our criteria for a public service body. These include[^1]:

- [.gov](https://get.gov/)
- [.gov.au](https://www.domainname.gov.au/)
- [.gc.ca](https://en.wikipedia.org/wiki/.gc.ca)
- [.gov.uk](https://www.gov.uk/apply-for-and-manage-a-gov-uk-domain-name)

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

When you're done, click `Submit`. This will generate and display the OAuth token that you need at the top of the page. The `Client key` and `Client secret` are the values you will need for creating the Tines credential. You don't have to write these down - you'll be able to access them from this menu whenever you need them.

#### Tines Credential

In order for your Tines workflow to authenticate to your Mastodon instance, it will need a credential. You can create one in your Tines team (everyone gets a team in Tines, even if they are an individual user) by expanding your team name and clicking on `Credentials` and then `New Credential` in the screen that appears. Pick `OAuth 2.0` from the dropdown menu.

Give your credential a suitable name, and then get ready to switch back and forth between this screen and your Mastodon `Development` screen as you set this up.

The Tines screen will give you a `Callback URL`. Copy this, click on your newly created `Application` in your Mastodon screen to edit it, and paste the `Callback URL` from Tines into the `Redirect URI` field there. While you are in the Mastodon screen, copy and paste both the `Client key` and `Client secret` from there into the `Client ID` and `Client secret` fields respectively in your Tines screen. When you're done, click `Save` in the Mastodon screen, but leave it open.

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

**NOTE:** You should replace `{my-mastodon-web_domain}`, `{my-credential}`, and `{my-email_address}` with your own values directly in the file **before** importing it. Remember to get rid of the braces `{}` as well!

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

[^1]: We are keen to add more - please [let us know](mailto:cunningpike@gmail.com) if there are any that should be added. Our worldview is regrettably US-centric, and there are likely many others that we have missed.
[^2]: This list is subject to change without notice, but we will keep our documentation of it, both here and on our instances, up to date.
[^3]: We haven't tried this ourselves so, if you do, [let us know](mailto:cunningpike@gmail.com) how it went!