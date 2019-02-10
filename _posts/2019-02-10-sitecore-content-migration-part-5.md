---
layout: post
title: Sitecore Content Migration, Part 5
tags: [sitecore, migration, export, xml]
---

Here's yet another installation in what is becoming a long running series. It's been a busy week, and I'm finally finding some time to get back to it. The goal this time is to start pumping out some XML filled with little tidbits of data I can feed to some other monster of a process that will transform it into something else entirely. Whether by magic or biology, it must be done.

When dealing with templates, I like to go with the Glass.Mapper/struct approach, and express each template with IDs and field names.  Consistency is tedious, but it helps, mostly, right? In this example I'll mock up an article of sorts, structured with a parent item which holds some common data, and child pages, which are actually rendered in the current site. As mention in [part 1](/2019-01-22-sitecore-content-migration-part-1) in this series, we have to convert to a single page article.  Since the child items are pages, they hold the metadata, but these articles will become single page, so the stakeholders agreed that the meta from the first page will be used.

<p class="mb-0">The basic template structure looks something like this:</p>

```cs
    public struct Templates
    {
        public struct Article
        {
            public const string TemplateIdString = "306D9178-1442-446B-8AFB-3FEEBAAE28CE";
            public static readonly ID TemplateId = new ID(TemplateIdString);

			
            public struct Fields
            {
                public static readonly ID ArticleTitle = new ID("849681EC-6FEC-4DB2-AE88-14CA1588F05D");
                public const string ArticleTitle_FieldName = "Article Title";

                public static readonly ID ArticleSubtitle = new ID("17C51CB8-DC4C-4122-BCCB-464D7C8D95BB");
                public const string ArticleSubtitle_FieldName = "Article Subtitle";

                public static readonly ID ArticleLinkSummary = new ID("49C36C93-F75F-4E38-BEBD-AFEE4E08CEA9");
                public const string ArticleLinkSummary_FieldName = "Article Link Summary";

                public static readonly ID ArticleImage = new ID("BEF85044-41FC-445D-A2B5-20F6D20E46DB");
                public const string ArticleImage_FieldName = "Article Image";

            }
        }

        public struct ArticlePage
        {
            public const string TemplateIdString = "4C2CFD94-4F18-4D39-ACE8-E0A69C6C6660";
            public static readonly ID TemplateId = new ID(TemplateIdString);

			
            public struct Fields
            {
                public static readonly ID ArticlePageTitle = new ID("67314EC3-C969-4B46-BC38-676CF6C6A360");
                public const string ArticlePageTitle_FieldName = "Article Page Title";

                public static readonly ID ArticlePageSubtitle = new ID("096335E7-1E04-40C4-8893-59C697B4B72E");
                public const string ArticlePageSubtitle_FieldName = "Article Page Subtitle";

                public static readonly ID Content = new ID("8E4C6F5F-E06D-49F0-B119-E91DCCD58D5B");
                public const string Content_FieldName = "Content";

            }
        }

        public struct Metadata
        {
            public const string TemplateIdString = "5DA93029-1EAB-41F9-8D3E-39DAC3F1A7E0";
            public static readonly ID TemplateId = new ID(TemplateIdString);

            public struct Fields
            {
                public static readonly ID MetaKeywords = new ID("440D814B-A651-4AC9-9EA5-345515E88B9D");
                public const string MetaKeywords_FieldName = "Meta Keywords";

                public static readonly ID MetaDescription = new ID("160FFC0C-2120-4D1A-8E72-5F910F611D67");
                public const string MetaDescription_FieldName = "Meta Description";

                public static readonly ID MetaTitle = new ID("F5B90D7B-85C4-4E57-8885-12B4313462B9");
                public const string MetaTitle_FieldName = "Meta Title";

                public static readonly ID Robots = new ID("D1D0726C-707E-4739-A54B-19DEB9380AC0");
                public const string Robots_FieldName = "Robots";

            }
        }
    }
```

<p class="mb-0">I'll break the templates into nodes, so we can do this from any level in the tree if need be. Now to run through an article and push things out in some sort of XML format...</p>

```cs
    public static class ArticleHelper
    {
        // Feed the monster!
        public static XmlDocument Export(Item item)
        {
            // Make sure it's the right type.
            if (item.TemplateID != Models.Templates.Article.TemplateId) return null;

            var xDoc = new XmlDocument();
            var rootNode = ArticleNode(item, xDoc);
            xDoc.AppendChild(rootNode);

            return xDoc;
        }

        // Separate this out by nodes.
        private static XmlElement ArticleNode(Item item, XmlDocument xDoc)
        {
            // Nodes, nodes, nodes!
            var ArticleNode = xDoc.CreateElement(nameof(Models.Templates.Article));

            var articleTitleNode = xDoc.CreateElement(nameof(Models.Templates.Article.Fields.ArticleTitle));
            articleTitleNode.InnerText = item[Models.Templates.Article.Fields.ArticleTitle];
            ArticleNode.AppendChild(articleTitleNode);

            var articleSubtitleNode = xDoc.CreateElement(nameof(Models.Templates.Article.Fields.ArticleSubtitle));
            articleSubtitleNode.InnerText = item[Models.Templates.Article.Fields.ArticleSubtitle];
            ArticleNode.AppendChild(articleSubtitleNode);

            var articleLinkSummaryNode = xDoc.CreateElement(nameof(Models.Templates.Article.Fields.ArticleLinkSummary));
            articleLinkSummaryNode.InnerText = item[Models.Templates.Article.Fields.ArticleLinkSummary];
            ArticleNode.AppendChild(articleLinkSummaryNode);

            var articleImageNode = xDoc.CreateElement(nameof(Models.Templates.Article.Fields.ArticleTOCImage));
            articleImageNode.InnerText = item[Models.Templates.Article.Fields.ArticleTOCImage];
            ArticleNode.AppendChild(articleImageNode);

            // Children are expected
            if (!item.HasChildren) return ArticleNode;

            // More specifically, an article page.
            var firstPage = item.Children.FirstOrDefault(c => c.TemplateID == Models.Templates.ArticlePage.TemplateId);
            if (firstPage == null) return ArticleNode;

            // Snag that meta.
            var metaKeywordsNode = xDoc.CreateElement(nameof(Models.Templates.Metadata.Fields.MetaKeywords));
            metaKeywordsNode.InnerText = item[Models.Templates.Metadata.Fields.MetaKeywords];
            ArticleNode.AppendChild(metaKeywordsNode);

            var metaDescriptionNode = xDoc.CreateElement(nameof(Models.Templates.Metadata.Fields.MetaDescription));
            metaDescriptionNode.InnerText = item[Models.Templates.Metadata.Fields.MetaDescription];
            ArticleNode.AppendChild(metaDescriptionNode);

            var metaTitleNode = xDoc.CreateElement(nameof(Models.Templates.Metadata.Fields.MetaTitle));
            metaTitleNode.InnerText = item[Models.Templates.Metadata.Fields.MetaTitle];
            ArticleNode.AppendChild(metaTitleNode);

            var robotsNode = xDoc.CreateElement(nameof(Models.Templates.Metadata.Fields.Robots));
            robotsNode.InnerText = item[Models.Templates.Metadata.Fields.Robots];
            ArticleNode.AppendChild(robotsNode);

            // Pluralization, eh?
            var pagesNode = xDoc.CreateElement($"{nameof(Models.Templates.ArticlePage)}s");

            // Everyone loves loops.
            foreach (Item itemChild in item.Children)
            {
                var articlePageNode = ArticlePageNode(xDoc, itemChild);

                pagesNode.AppendChild(articlePageNode);
            }

            ArticleNode.AppendChild(pagesNode);

            return ArticleNode;
        }

        // Same game, new node.
        private static XmlElement ArticlePageNode(XmlDocument xDoc, Item itemChild)
        {
            // So much nodez.
            var ArticlePageNode = xDoc.CreateElement(nameof(Models.Templates.ArticlePage));

            var articlePageTitleNode = xDoc.CreateElement(nameof(Models.Templates.ArticlePage.Fields.ArticlePageTitle));
            articlePageTitleNode.InnerText = itemChild[Models.Templates.ArticlePage.Fields.ArticlePageTitle];
            ArticlePageNode.AppendChild(articlePageTitleNode);

            var articlePageSubtitleNode =
                xDoc.CreateElement(nameof(Models.Templates.ArticlePage.Fields.ArticlePageSubtitle));
            articlePageSubtitleNode.InnerText = itemChild[Models.Templates.ArticlePage.Fields.ArticlePageSubtitle];
            ArticlePageNode.AppendChild(articlePageSubtitleNode);

            var contentNode = xDoc.CreateElement(nameof(Models.Templates.ArticlePage.Fields.Content));
            // Oh my god, look, it's that other thing, you know.
            var html = HtmlHelper.CleanImageTags(itemChild,
                Models.Templates.ArticlePage.Fields.Content_FieldName);
            contentNode.InnerText = WebUtility.HtmlEncode(html);
            ArticlePageNode.AppendChild(contentNode);
            
            return ArticlePageNode;
        }
    }
```

That's pretty much what's going to be happening from here on out for exporting.  Lots of templates, lots of XML, and some more templates & XML.  I imagine the mapping back from XML to items will also look al ot like this, but we'll see if that holds true next time around.

As we've been taught by so many kittens, hang in there.

-Ben