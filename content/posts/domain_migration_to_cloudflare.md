+++
date = '2026-04-02T16:00:00-05:00'
draft = false
title = 'I Moved My Blog to a Custom Domain'
tags = ["hugo", "cloudflare", "github-pages"]
categories = ["devops"]
description = "Migrating mrmcmuffinz.github.io to mrmcmuffinz.dev with Cloudflare DNS, and the small surprises along the way."
layout = "post"
+++

# I Moved My Blog to a Custom Domain (and My AI Assistant Made Up a URL)

Welcome back friends. This post is short and sweet. I migrated my site from `mrmcmuffinz.github.io` to `mrmcmuffinz.dev` today, and I wanted to walk through the steps I took to complete the migration.

## Why Bother

The `.github.io` domain works fine. GitHub Pages is free, the deployment pipeline was already set up, and nobody was complaining about the URL. But I've been building out more projects, thinking about professional presence, and a custom domain is one of those things where the effort-to-payoff ratio is heavily in my favor. I picked up `mrmcmuffinz.dev` through Cloudflare for a few dollars a year, and the migration itself took about an hour of actual work. There's also a practical reason: owning the domain means I'm not locked to GitHub Pages forever. If I ever want to move hosting to Cloudflare Pages, a VPS, or something else entirely, the URL stays the same. Visitors, search engines, and bookmarks all keep working. With a `.github.io` domain, moving off GitHub Pages means starting over on every link anyone has ever shared.

## What Actually Changed

This is a Hugo site using the Blowfish theme, deployed to GitHub Pages on push to `main`. The migration touched three layers: DNS, the repository, and third-party services.

Here's what the setup looked like before:

{{< mermaid >}}
graph LR
    A[User] -->|visits| B[mrmcmuffinz.github.io]
    B --> C[GitHub Pages]
{{< /mermaid >}}

And after:

{{< mermaid >}}
graph LR
    A[User] -->|visits| B[mrmcmuffinz.dev]
    B -->|CNAME flattening| C[Cloudflare DNS]
    C -->|resolves to| D[GitHub Pages]
    E[mrmcmuffinz.github.io] -->|301 redirect| B
{{< /mermaid >}}

On the DNS side, I added two CNAME records in Cloudflare. One for the apex domain (`mrmcmuffinz.dev` pointing to `mrmcmuffinz.github.io`) and one for `www`. Cloudflare has a feature called CNAME flattening that lets you use a CNAME record on an apex domain, which normally isn't allowed by the DNS spec. Cloudflare resolves it behind the scenes into the appropriate A/AAAA records, so it's spec-compliant from the perspective of anyone querying the DNS. This is simpler than hardcoding GitHub's four IPv4 and four IPv6 addresses, and it means if GitHub ever changes their Pages infrastructure IPs, I don't have to update anything. One thing that tripped me up briefly: Cloudflare defaults new DNS records to "Proxied" mode, which routes traffic through Cloudflare's CDN and applies their SSL certificate. That conflicts with GitHub Pages, which provides its own SSL certificate and needs DNS to resolve directly to their servers. The fix is simple, just toggle each record to "DNS only", but it's the kind of default that could leave you debugging SSL errors if you don't catch it.

In the repository, I updated the Hugo `baseURL` in `config/_default/hugo.toml`, created a `static/CNAME` file (Hugo copies everything in `static/` to the site root at build time, and GitHub Pages reads the CNAME file to know which custom domain to serve), and then hunted down every hardcoded reference to the old domain. That turned out to be eleven files: the Hugo config, the LinkedIn RSS workflow, the contact form redirect, the email subject line in the contact form shortcode, the privacy policy, two blog posts with internal cross-links, the LinkedIn post tracker file, the README, and the CLAUDE.md project docs. Nothing complicated, just a thorough find-and-replace with a `grep` pass afterward to make sure nothing was missed. On the GitHub side, I went to the repo's Settings, Pages, and set the custom domain to `mrmcmuffinz.dev`. Once DNS propagated and GitHub verified the domain, I enabled "Enforce HTTPS." GitHub automatically sets up a 301 redirect from `mrmcmuffinz.github.io` to the new domain, so any existing links or bookmarks continue to work without any extra configuration.

## The Third-Party Cleanup

{{< mermaid >}}
graph LR
    A[mrmcmuffinz.dev] --> B[Google Analytics]
    A --> C[Disqus]
    A --> D[Formspree]
{{< /mermaid >}}

Three external services needed attention:

1. Google Analytics was the easiest. The GA4 measurement ID (`G-E2DJHQS5DF`) is domain-agnostic, it tracks based on the ID embedded in your site's code, not the domain name configured in the dashboard. I updated the stream URL in GA4 Admin for cleanliness, but tracking would have continued working either way.

2. Formspree was similarly straightforward. I added `mrmcmuffinz.dev` to the allowed domains for my form endpoint so the contact form would continue accepting submissions from the new origin.

3. Disqus was the one that caught me off guard. After the migration, every post showed zero comments. Disqus maps comment threads to exact URLs, so when the domain changed from `mrmcmuffinz.github.io/posts/some-post/` to `mrmcmuffinz.dev/posts/some-post/`, Disqus treated them as completely different pages. The fix was straightforward once I found it: Disqus has a Domain Migration Tool in the admin panel where you enter the old domain and the new domain, and it remaps all existing threads automatically. But if I hadn't checked the comments after deploying, I might not have noticed for days. It's the kind of thing that's obvious in hindsight and easy to miss in the moment.

## The Result

`mrmcmuffinz.dev` is live. HTTPS works. The old `.github.io` domain redirects cleanly. Comments are intact. Analytics are tracking. The contact form works. The LinkedIn RSS workflow will pick up the new feed URL on its next scheduled run. Total time from starting to deploying was about an hour, with another few minutes for the Disqus migration tool to process.

If you're running a Hugo site on GitHub Pages and thinking about switching to a custom domain, it's genuinely not that complicated. Buy the domain, add two DNS records, update your `baseURL`, grep for hardcoded URLs, set the custom domain in GitHub's settings, and check your third-party integrations. The hardest part is remembering to check all the places where your old domain might be hiding.

Well, that's all. See you in the next one.

**P.S.** I used Claude Code to handle the repository changes for this migration and as a sounding board while writing this post. The ideas, the fact-checking, and the final edits are mine.
