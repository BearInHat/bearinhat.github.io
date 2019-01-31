---
layout: post
title: Sitecore Content Migration, Part 3
tags: [sitecore, migration, export]
---

Welcome back, and thanks for keeping at this with me. I think I'll be getting a bit more into code this time, so brace yourselves!

Following up on the last post I'd like to finish building the image tags, so let's start with pulling finding the images in Sitecore, and getting out the attributes we need.  I know from auditing the current items the image sources have five different formats they could be in, and IIS rewrites are taking care of the rest.
1. Current Site/path
2. Previous Site/path
3. /Relative Path Without Media Prefix
4. /Media Prefix/Path
5. External Path

This will make it a bit easier to handle resolving the media paths, since there's a limited set of parameters to deal with.

At this point I'll head over to Visual Studio, spin up a console project, pull in the Sitecore nuget package for Kernel, and copy over a set of clean configs from the 8.2 installation zip.  After copying the content from the web.config to the app.config, and updating the connection strings, I'm ready to go.

<p class="mb-0">Let's add a folder for Utilities (Helpers) and create a class for cleansing HTML. Let's also bring in the functionality from the previous post, something along these lines.</p>

```cs
public static class HtmlHelper{
    // Existing Fields
    // New Fields
    public static string CleanImageTags(Item item, string field){
        Assert.NotNull(item, nameof(item));
        Assert.NotNull(field, nameof(field));
        
        if(!item.Fields(field).HasValue) return string.empty;
        var html = item[field];
        // Existing Functionality
        // New Functionality
        return html;
    }
}
```

<p class="mb-0">Since extra manipulation is needed on the images, it would be easiest to iterate the Matches, and handle them one at a time.  Using the second group, a couple checks, and some cleanup, I can find the images in Sitecore, as long as they aren't external images. I'm also going to update the image replacement field from a Regex statement to a formatting string, since a bit more needs to be done. So let's set up those variables.</p>

```cs
    // New image replacement...
    private const string ImgSrcReplace = "<img {0} src=\"{1}\" data-src=\"{2}\" {3}>";
    // Our Site Variations
    private const string PreviousSite = "previoussite.org";
    private const string CurrentSite = "currentsite.com";
    // Format Characters
    private const string Slash = "/";
    // Prefixes for image paths in Sitecore
    private const string MediaVariation = "/~";
    private const string MediaStandard = "/-";
    // Sitecore Tree prefix
    private const string SitecoreMediaPath = "/sitecore/media library/site/images";
    // src prefix
    private const string SiteMediaPathPrefix = "/-/media/site/images";
    // placeholder image
    private const string PlaceholderImg = "/-/media/project/site/placeholders/small.png";
```

<p class="mb-0">This covers all of the formats that we've found, and should be enough to handle what we're about to do. I didn't do it above, so I'll add a check to make sure there's a match at all, then go through the collection.</p>

```cs
    var matches = ImgSrcReg.Matches(html);
    if (matches.Count < 1) return html;

    // Iterate images with src tags
    foreach (Match match in matches)
    {
        // No src, nothing to replace.
        if (string.IsNullOrWhiteSpace(match.Groups[2].Value)) continue;
        // Hit with blunt object!
    }
```

<p class="mb-0">Just as I was starting into this, I realized I've been forgetting about existing query strings, so I'll try to work that into it.  Here's that blunt object... (inside loop)</p>

```cs
    // Break apart the src url.
    var srcParts = match.Groups[2].Value.Split('?');
    // Normalize url format.
    var imagePath = srcParts[0].Replace("\\", Slash);
    // Prep query strings
    var hasQuery = srcParts.Length > 1;
    var query = hasQuery ? srcParts[1] : "";
    var heightWidthQuery = "";
    // Flag for check in sitecore.
    var isExternal = false;

    // Check if image is from our site, and make relative.
    if (imagePath.Contains(PreviousSite))
    {
        var siteIndex = imagePath.IndexOf(PreviousSite, StringComparison.OrdinalIgnoreCaseOrdinal);
        imagePath = imagePath.Substring(siteIndex + PreviousSite.Length);
    }
    else if (imagePath.Contains(CurrentSite))
    {
        var siteIndex = imagePath.IndexOf(CurrentSite, StringComparison.OrdinalIgnoreCase);
        imagePath = imagePath.Substring(siteIndex + CurrentSite.Length);
    }
    // Not from our site, sot trim protocol.
    else if (imagePath.StartsWith("http", StringComparison.OrdinalIgnoreCase))
    {
        var colonIndex = imagePath.IndexOf(":", StringComparison.OrdinalIgnoreCase);
        imagePath = imagePath.Substring(colonIndex);
        // Ignore Sitecore lookup.
        isExternal = true;
    }
```

<p class="mb-0">At this point we have images in a format that will make finding them in Sitecore easier to handle, and we know if we should even check the images. Moving on... (still in the loop)</p>

```cs
    // Normalize Sitecore media format, and get image details
    if (!isExternal)
    {
        // Ensure leading slash
        if (!imagePath.StartsWith(Slash))
        {
            imagePath = "/" + imagePath;
        }
        // Standardize path 
        if (imagePath.StartsWith(MediaVariation))
        {
            imagePath = imagePath.Replace(MediaVariation, MediaStandard);
        }
        // Remove Sitecore media Prefix
        // Needed for Item Lookup
        if (imagePath.StartsWith(SiteMediaPathPrefix))
        {
            imagePath = imagePath.Substring(SiteMediaPathPrefix.Length - 1);
        }
        // Search for the MediaItem
        var lastPeriod = imagePath.LastIndexOf('.');
        var sansExtension = lastPeriod > 0 ? imagePath.Substring(0, lastPeriod) : imagePath;
        MediaItem image = item.Database.GetItem($"{SitecoreMediaPath}{sansExtension}");
        // Make sure we have it...
        if (image != null)
        {
            // TODO: Maybe Export the images?
            // Get media info and build query
            // Height and Width are expected, but just in case.
            var height = image.InnerItem.Fields["Height"].HasValue ? $"h={image.InnerItem["Height"]}" : "";
            var width = image.InnerItem.Fields["Width"].HasValue ?  $"w={image.InnerItem["Width"]}" : "";
            var hwAmp = string.IsNullOrWhiteSpace(height) || string.IsNullOrWhiteSpace(width) ? "" : "&";
            heightWidthQuery = $"{height}{hwAmp}{width}";
        }
        // Add the queries together.
        var queryAmp = string.IsNullOrWhiteSpace(heightWidthQuery) || string.IsNullOrWhiteSpace(query) ? "" : "&";
        query = $"{heightWidthQuery}{queryAmp}{query}";
    }

    // Rebuild the image path
    query = string.IsNullOrWhiteSpace(query) ? "" : $"?{query}";
    imagePath = $"{SiteMediaPathPrefix}{imagePath}{query}";
    // Create placeholder path
    // TODO: Do we need the previous query for the placeholder?
    var placeholderSrc = $"{PlaceholderImg}{query}";
    // Format the new image tag.
    var imgTag = string.Format(ImgSrcReplace, match.Groups[1].Value, placeholderSrc, imagePath,
        match.Groups[3].Value);
    // Replace this image in the html (Group 0 is always full match.)
    html = html.Replace(match.Groups[0].Value, imgTag);
    // loop loop loop
```

Now that's a blunt object if I've ever seen one, but it is taking care of business for now, so I won't complain.  Next time I think I'll do something with exporting the images, while we have a hook on each of them.  Later on down the line I'll put it all together into something a little cleaner.

Little by little...

-Ben