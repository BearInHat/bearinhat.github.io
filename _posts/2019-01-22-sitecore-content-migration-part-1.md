---
layout: post
title: Sitecore Content Migration, Part 1
tags: [sitecore, helix, migration]
---

Currently I'm working on a project to rewrite a rich text blob heavy Sitecore site in a Helix friendly fashion.  Along with this there's also a requirement to use lazy loading for images, videos, and other media content. It's also expected that page URLs will remane the same were possible, to avoid any hits to SEO.  This is going to be a bit of a challenge though, since they want to go to a single page format for media, even though it's currently in a multi page format.

The approach will be to cleanse the existing HTML in the rich text fields and place them back, blob-style, into the new templates, then have editors use a full Helix style editor experience after that.  The cleansing is promising to be a blast, because a lot of the legacy content was initially migrated to Sitecore from a previous CMS, and there's stale sources, in the HTML tags, pointing to the old site! This is all being managed with rewrites currently, but while I'm here I might as well deal with it, so I'll also have to capture and cleanse these links to be a relative path.

Further more, to make their overall experience and transition into this new mindset of managing content as easy as possible, I'll try to make some good use of branches to help generate new content going forward.

In the next few (many?) posts I'll try getting into some code or examples as I try out some ideas.

Stay tuned for more!

-Ben

