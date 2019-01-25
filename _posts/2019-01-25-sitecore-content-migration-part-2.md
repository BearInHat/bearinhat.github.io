---
layout: post
title: Sitecore Content Migration, Part 2
tags: [sitecore, migration, regex]
---

Alright, let's see if we can start making some steps forward. Looking over the current Sitecore implementation, and the tasks at hand, I'm going to start playing around with cleaning some HTML for export.  I'll tackle it little by little, starting with images, and converting the tags to a usable format.

To start I'll head over to [.NET Fiddle](https://dotnetfiddle.net/). Over the past few years, I've really been enjoying using it as playground for basic C#.  It's a lot more helpful then spinning up a console app just to test things.

<p class="mb-0">As far as lazy loading goes, I'm a fan of <a href="https://github.com/ApoorvSaxena/lozad.js">Lozad</a>.  That being said, the goal is to move from:</p>
```html
<img attributes="" src="currentPath" class="someClass" otherAttributes="" />
```
<p class="my-0">to:</p>
```html
<img attributes="" src="placeHolderPath" data-src="currentPath" class="someClass lozad" otherAttributes="" />
```
As much as some people frown upon it, I'm going to do this with a little help from Regex. As long as the patterns aren't too complicated, I think this will be a decent approach.  I also know the content has been carefully curated for some time, so the likelihood of malformed tags is low.

<p class="mb-1">Instead of trying to make some monstrous expression, we can handle this in a few expressions. The first one will be to find images with a valid src attribute, and the second can be a pattern for replacement.  We'll also need some mock html to test against.</p>
```cs
var imgSrcMatch = "<img(\\s*[^>]*?)src\\s*=\\s*['\"]([^'\"]*)['\"](\\s*?[^>]*?)>";
var imgSrcReplace = "<img$1src=\"\" data-src=\"$2\"$3>";
var html = "<div><div><img alt=\"Image 1\" src=\"/path/image1.jpg\" style=\"border-width: 0px; border-style: solid;\" /><img src=\"http://site/path/image2.jpg\" alt=\"Image 2\" class=\"image2\" /><img class=\"image3\" src='/image3.jpg' alt=\"Image 3\" /></div><div><img src=\"image4.jpg\" /><img src=\"image5.jpg /><img class=\"image6\" /></div></div>";
```
<p class="mb-1">The match basically says, img tags that aren't immediately closed, have some optional content, then a src attribute with a value, some more optional content, and a close tag. The replace is simply going to take the three groups from above, and rebuild the tag.  The mock html has six images, but the last two shouldn't match.  Let's new up a Regex and do some testing.</p>
```cs
var imgSrcReg = new Regex(imgSrcMatch);

Console.WriteLine("Original:");
foreach(Match match in imgSrcReg.Matches(html)){
    Console.WriteLine("Orig: " + match.Groups[0].ToString());
}
html = Regex.Replace(html, imgSrcMatch, imgSrcReplace);

Console.WriteLine("");
Console.WriteLine("Transformed:");
foreach(Match match in imgSrcReg.Matches(html)){
    Console.WriteLine(match.Groups[0].ToString());
}
```
<p class="mb-0 font-weight-bold">Output:</p>
```plaintext
Original:
<img alt="Image 1" src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="http://site/path/image2.jpg" alt="Image 2" class="image2" />
<img class="image3" src='/image3.jpg' alt="Image 3" />
<img src="image4.jpg" />
<img src="image5.jpg /><img class="image6" />

Transformed:
<img alt="Image 1" src="" data-src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="" data-src="http://site/path/image2.jpg" alt="Image 2" class="image2" />
<img class="image3" src="" data-src="/image3.jpg" alt="Image 3" />
<img src="" data-src="image4.jpg" />
<img src="" data-src="image5.jpg />
```
<p class="mb-1">So 5 and 6 are matching in an unexpected way.  The group for the src value is letting the match extend beyond the closing bracket. A quick tweak to include '>', and another round of testing.</p>
```cs
var imgSrcMatch = "<img(\\s*[^>]*?)src\\s*=\\s*['\"]([^'\">]*)['\"](\\s*?[^>]*?)>";
```
<p class="mb-0 font-weight-bold">Round 2:</p>
```plaintext
Original:
<img alt="Image 1" src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="http://site/path/image2.jpg" alt="Image 2" class="image2" />
<img class="image3" src='/image3.jpg' alt="Image 3" />
<img src="image4.jpg" />

Transformed:
<img alt="Image 1" src="" data-src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="" data-src="http://site/path/image2.jpg" alt="Image 2" class="image2" />
<img class="image3" src="" data-src="/image3.jpg" alt="Image 3" />
<img src="" data-src="image4.jpg" />
```

Much better!

<p class="mb-1">Let's use a similar approach for including lozad in the class, both for images with and without a class attribute.</p>
```cs
// Like the src match, but for class, and with a positive lookahead to guarantee the src attribute
var imgClsMatch = "<img(?=[^>]+src=['\"])(\\s*[^>]*?)class\\s*=\\s*['\"]([^'\">]*)['\"](\\s*?[^>]*?)>";
// Append lozad to class
var imgClsReplace = "<img$1class=\"$2 lozad\"$3>";
// Positive lookahead on src, negative lookahead on class, only one group
var imgNoClsMatch = "<img(?=[^>]+src=['\"])(?![^>]+class=['\"])(.*?)>";
// Add class
var imgNoClsReplace = "<img class=\"lozad\"$1>";
// After src replacement
html = Regex.Replace(html, imgClsMatch, imgClsReplace);
html = Regex.Replace(html, imgNoClsMatch, imgNoClsReplace);
```
<p class="mb-0 font-weight-bold">Altogether now!</p>
```plaintext
Original:
<img alt="Image 1" src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="http://site/path/image2.jpg" alt="Image 2" class="image2" />
<img class="image3" src='/image3.jpg' alt="Image 3" />
<img src="image4.jpg" />

Transformed:
<img class="lozad" alt="Image 1" src="" data-src="/path/image1.jpg" style="border-width: 0px; border-style: solid;" />
<img src="" data-src="http://site/path/image2.jpg" alt="Image 2" class="image2 lozad" />
<img class="image3 lozad" src="" data-src="/image3.jpg" alt="Image 3" />
<img class="lozad" src="" data-src="image4.jpg" />
```

This looks like a good start! I'm sure I'm overlooking a few things, but I'll deal with them as they come.  Next time I'll work on doing this with a Sitecore Item. I also want to include some parameters on the source paths so our placeholder image takes up the same space, so I'll have to resolve the images in Sitecore and get back those values.

Until then...

-Ben