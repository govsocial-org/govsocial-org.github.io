---
#template: home.html
title: GOVSocial Documentation
---

# GOVSocial Documentation

One of our goals from the outset of the project was to contribute to the Fediverse community by providing comprehensive documentation of our implementation and operation journey for others who wanted to deploy on GKE in a similar way that we did. As part of our initial research, we found few examples of instances that had been built entirely on Google Kubernetes Engine (GKE) and other Google Cloud Platform services.

We had a couple of requirements for our document site:

- **It needed to use [MarkDown](https://www.markdownguide.org/)**. We wanted to use an open standard, and generate our site from easily portable text documents, in case we wanted to change our hosting platform.
- **It needed to be a static site**. We didn't want a CMS with proprietary databases or plugins.
- **It needed to be independently hosted, but not self-hosted**. We wanted the freedom of hosting the site outside of an proprietary online CMS ecosystem, without the headaches of implementing, securing, and managing our own HTTP server.

## MkDocs

After some research, we settled on [MkDocs-Material](https://squidfunk.github.io/mkdocs-material/) as our static content generator. [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/) uses it, and it suits the creation and maintenance of technical documentation very well. It is powerful, but also simple to get started with[^1].

We primarily use Chromebooks as our workstations, which presents a challenge when it comes to installing IDE tools locally. Enter [GitHub Codespaces](https://docs.github.com/en/codespaces/overview)[^2]. Every personal and organizational repository on GitHub comes with a basic amount (60 hours and 15GB per month) of Codespaces, allowing any device with Internet access to run an IDE of choice.

Even better, you can [create your Codespace from an existing GitHub repo](https://docs.github.com/en/codespaces/developing-in-codespaces/creating-a-codespace-for-a-repository), which is exactly what we did.

## GitHub Repository

We forked the [MkDocs-Material](https://github.com/squidfunk/mkdocs-material) into [our own repo](https://github.com/govsocial-org/govsocial-org.github.io/), named so that we could use [GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site) to host it.

From our repo, we created our Codespace, following [these instructions](https://squidfunk.github.io/mkdocs-material/getting-started/#with-git) from MkDocs's own [excellent documentation](https://squidfunk.github.io/mkdocs-material/). Notice that we ***forked*** the repo rather than ***cloning*** it - we wanted to be able to sync with the original repo from time to time to keep things up to date[^3].

Your GitHub Codespace will prompt you for some Python extensions, which you should install, but apart from that, getting it up and running is very straightforward. We have left it as an exercise for you, dear reader, to learn [MarkDown](https://www.markdownguide.org/basic-syntax/) and [MkDocs-Material](https://squidfunk.github.io/mkdocs-material/reference/) from their own excellent documentation sites.

You can preview your site as you create it by typing `mkdocs serve` in your Codespace terminal, and clicking on `Open in Browser` in the dialog that appears.

## GitHub Pages

As mentioned above, we decided to go all in with GitHub for the site and use [GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site) to host it. To create a GitHub pages site, all you need to do is name your repo using the `[user|organization].github.io` naming convention. Your repo can either be public or private, depending on your preference.

Integrating with MkDocs website deployment is a snap. Commit your local changes to your repo, and then run `mkdocs gh-deploy --force` from your Codespace. This will force push everything in the `docs` (the MkDocs default) directory into a `gh-pages` branch in your repo.

Configure your GitHub Pages (in Settings) to publish from the `gh-pages` branch of your repo, and you should see your content published at `https://[user|organization].github.io` within a minute or so.

**Note:** The MkDocs documentation suggests examples of ways of automating the deployment of your site each time you commit changes to your repo. We opted to deploy manually, as we wanted to publish to the public site when we wanted to, and not every time someone committed a change. YMMV.

## Custom Domain

You will [remember](/building/mastodon/#load-balancer) that we created a load balancer to direct all paths other than the one needed for WebFinger to `docs.govsocial.org`. Using this for our GitHub Pages site was striaghtforward as well:

- Create a `CNAME` entry in your Cloud DNS record set that redirects your custom documentation hostname (`docs.govsocial.org`, in our case) to your GitHub Pages hostname (`[user|organization].github.io`).
- Next, **and this is an important if minor step that we initially missed**, create a file called `CNAME` in at the top of your `docs` directory structure. This file should contain your custom domain in it, and nothing else, like this:
```bash title="CNAME"
docs.govsocial.org
```
When you complete the next step, GitHub automatically creates this file in the `gh-pages` branch of your repo, and it will get clobbered by a deployment if you haven't added it to your local copy.
- Finally, go to the Settings for your GitHub pages, and enter your custom domain in the appropriate place.

Once this all syncs up, and if you have been following all the steps we took, your naked domain should redirect to your custom docs domain, and that in turn will redirect to your GitHub pages site!

[^1]: Full disclosure: we unabashedly copied a lot of what the Cookbook site uses for their site - imitation is the sincerest form of flattery :-).
[^2]: Hat-tip to Elan@PublicSquare.global for suggesting this.
[^3]: It turns out that the MkDocs-Material repo is **very** active, and we quickly ended up in a situation where we need to merge changes rather than automatically syncing them. This is on our TO-DO list.