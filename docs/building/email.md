---
#template: home.html
title: Transactional Email
---

# Transactional Email

Another service that most Fediverse platforms use in common is transactional email. These are the emails that allow users to confirm their email address, reset passwords, and receive the notifications they set up for themselves.

This was the one service we were unable to host with Google, as they don't really provide a [Mail Transfer Agent (MTA)](https://www.techopedia.com/definition/1691/message-transfer-agent-mta) that operates at scale. Personal GMail accounts do have [limited mail sending capacity](https://support.google.com/a/answer/166852), but a Fediverse instance of any size will rapidly exceed that, and may have the unfortunate result of suspending your account.

After a couple of failed attempts at persuading a couple of the well-known commercial transactional mail services of our bona fides, we settled on [Amazon AWS Simple Email Service (AWS SES)](https://aws.amazon.com/ses/). It has generous daily sending limits at a very reasonable cost.

## Setting up AWS SES

You will need to set up an AWS account, and create an SES instance. Your new SES account will be in a sandox (meaning it can only send mail to its own identities) until you apply to move out of the sandbox. This process takes about a day, so we recommend starting early, while you work on some of the other preparation steps.

While you are waiting, take the time to verify your domain for [DKIM](https://www.dkim.org/), and set up a `Custom From Domain` (we use `notifications.govsocial.org` for ours). The console will guide you through the steps for creating the necessary DNS entries. Again, this can take a while to propagate, so get this going early.

Use the `SMTP Settings` menu in the SES console to create an SMTP credential in the AWS IAM. This will provide you with an account name and authentication key that you will need for your platform configuration.

!!! Warning
    **Save these somewhere safe - you will only be shown them once at creation time**.

That menu will also provide the configuration settings for the SMTP server that you will need.