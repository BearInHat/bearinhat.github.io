---
layout: post
title: Sitecore Content Migration, Part 4
tags: [sitecore, migration, export]
---

I guess it's time to do something for exporting images, so this will probably be a short post since this is pretty straight forward. Maybe not completely straight forward, because there's two options for dealing with this. One is to actually stream the media out to a path, then, on the more complicated side of things, will be importing the images back to the new site.  The other would be to dump (serialize) each media item, then load (deserialize) them on the way back in.

With both options there is a bit of complication because the site structure is changing. This means the paths of the items are not longer valid, and the parent item will no longer be the same.  With serialized items, that means parsing the item file to update some of the meta to make sure path and parent ID match.  If I go with streaming and creating items, the path is less of a concern, but the image IDs change, so items currently referencing the images need to be updated.

<p class="mb-0">Both of the methods are easy to create as far as the export side of things, so I'll just go ahead and do both for now, and see which is easier to work with when everything comes together.</p>

```cs
    public static class ImageHelper
    {
        // TODO: Set from config?
        private const string ExportPath = "C:\\Temp";

        // Download the item.
        public static void Download(MediaItem mediaItem)
        {
            // Ensure Item
            if(mediaItem == null) return;
            // Get a path and make sure the directory exists.
            var fullPath = GetExportPath(mediaItem);
            EnsurePath(fullPath);
            // Get Stream
            var media = MediaManager.GetMedia(mediaItem);
            var stream = media.GetStream();
            // Download
            using (var targetStream = File.OpenWrite(Path.Combine(fullPath, $"{mediaItem.Name}.{mediaItem.Extension}")))
            {
                stream.CopyTo(targetStream);
                targetStream.Flush();
            }
        }

        // Serialize the item.
        public static void Serialize(MediaItem mediaItem)
        {
            // Ensure Item
            if(mediaItem == null) return;
            // Get a path and make sure the directory exists.
            var fullPath = GetExportPath(mediaItem);
            EnsurePath(fullPath);
            // Set final item path
            fullPath = $"{fullPath}{mediaItem.Name}.item";
            // Serialize
            Manager.DumpItem(fullPath, mediaItem);
        }

        // Build the path.
        private static string GetExportPath(MediaItem mediaItem)
        {
            return Path.GetFullPath(Path.Combine(Path.Combine(ExportPath,
                $"{mediaItem.MediaPath.Substring(1).Substring(0, mediaItem.MediaPath.IndexOf($"/{mediaItem.Name}", StringComparison.Ordinal))}")));
        }

        // Create the path if it doesn't exist.
        private static void EnsurePath(string path)
        {
            if (!Directory.Exists(path))
            {
                Directory.CreateDirectory(path);
            }
        }
    }
```

That's pretty much it for that process.  Like I said they're both pretty simple to implement.  I might return a bool from then for a success flag, but that's about it. Next time I think I'll start building out an file, probably xml, to put all the existing and cleaned data into, so we have a structure for the import later on.

The journey continues.

-Ben