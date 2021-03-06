uploadfs
========

<a href="http://apostrophenow.org/"><img src="https://raw.github.com/punkave/uploadfs/master/logos/logo-box-madefor.png" align="right" /></a>

uploadfs copies files to a web-accessible location and provides a consistent way to get the URLs that correspond to those files. uploadfs can also resize, crop and autorotate uploaded images. uploadfs includes both S3-based and local filesystem-based backends and you may supply others. The API offers the same conveniences with both backends, avoiding the most frustrating features of each:

* Parent directories are created automatically as needed (like S3)
* Content types are inferred from file extensions (like the filesystem)
* Files are always marked as readable via the web (like a filesystem + web server)
* Images can be automatically scaled to multiple sizes
* Images can be cropped
* Images are automatically rotated if necessary for proper display on the web (i.e. iPhone photos with rotation hints are right side up)
* Image width, image height and correct file extension are made available to the developer
* Non-image files are also supported
* Web access to files can be disabled and reenabled

You can also remove a file if needed.

It is possible to copy a file back from uploadfs, but there is no API to retrieve information about files in uploadfs. This is intentional. *Constantly manipulating directory information is much slower in the cloud than on a local filesystem and you should not become reliant on it.* Your code should maintain its own database of file information if needed, for instance in a MongoDB collection. Copying the actual contents of the file back may occasionally be needed however and this is supported.

## Requirements

You need:

* A "normal" filesystem in which files stay put forever, *OR* Amazon S3, OR a willingness to write a backend for something else (look at `s3.js` and `local.js` for examples).

* [Imagemagick](http://www.imagemagick.org/script/index.php), if you want to use `copyImageIn` to automatically scale images; OR, on Macs, the [imagecrunch](http://github.com/punkave/imagecrunch) utility; OR a willingness to write a backend for something else (look at `imagemagick.js` and `imagecrunch.js` for examples).

* A local filesystem in which files stay put at least during the current request, to hold temporary files for Imagemagick's conversions. Heroku and most other cloud environments can keep a file alive at least that long, and of course so does any normal, boring VPS or dedicated server.

Note that Heroku includes Imagemagick. You can also install it with `apt-get install imagemagick` on Ubuntu servers. The official Imagemagick binaries for the Mac are a bit busted as of this writing, but macports or homebrew can install it. Or, you can use [imagecrunch](http://github.com/punkave/imagecrunch), a fast, tiny utility that uses native MacOS APIs.

## API Overview

Here's the entire API:

* The `init` method passes options to the backend and invokes a callback when the backend is ready.

* The `copyIn` method takes a local filename and copies it to a path in uploadfs. (Note that Express conveniently sets us up for this by dropping file uploads in a temporary local file for the duration of the request.)

* The `copyImageIn` method works like `copyIn`. In addition, it also copies in scaled versions of the image, corresponding to the sizes you specify when calling `init()`. Information about the image is returned in the second argument to the callback.

* If you wish to crop the image, pass an options object as the third parameter to `copyImageIn`. Set the `crop` property to an object with `top`, `left`, `width` and `height` properties, all specified in pixels. These coordinates are relative to the original image. **When you specify the `crop` property, both the "full size" image copied into uploadfs and any scaled images are cropped.** The uncropped original is NOT copied into uploadfs. If you want the uncropped original, be sure to copy it in separately. The `width` and `height` properties of the `info` object passed to your callback will be the cropped dimensions.

* The default JPEG quality setting for scaled-down versions of your image is `80`. This avoids unacceptably large file sizes for web deployment. You can adjust this via the `scaledJpegQuality` option, either when initializing uploadfs or when calling `copyImageIn`.

* The `copyOut` method takes a path in uploadfs and a local filename and copies the file back from uploadfs to the local filesystem. This should be used only rarely. Heavy reliance on this method sets you up for poor performance in S3. However it may be necessary at times, for instance when you want to crop an image differently later. *Heavy reliance on copyOut is a recipe for bad S3 performance. Use it only for occasional operations like cropping.*

* The `remove` method removes a file from uploadfs.

* The `getUrl` method returns the URL to which you should append uploadfs paths to fetch them with a web browser.

* The `disable` method shuts off web access to a file. Depending on the storage backend it may also block the `copyOut` method, so you should be sure to call `enable` before attempting any further access to the file.

* The `enable` method restores web access to a file.

* The `getImageSize` method returns the currently configured image sizes.

* The `identifyLocalImage` method provides direct access to the `uploadfs` functionality for determining the extension, width, height and orientation of images. Normally `copyIn` does everything you need in one step, but this method is occasionally useful for migration purposes.

## Working Example

For a complete, very simple and short working example in which a user uploads a profile photo, see `sample.js`.

Here's the interesting bit. Note that I do not supply an extension for the final image file, because I want to let Imagemagick figure that out for me.

    app.post('/', function(req, res) {
      uploadfs.copyImageIn(req.files.photo.path, '/profiles/me', function(e, info) {
        if (e) {
          res.send('An error occurred: ' + e);
        } else {
          res.send('<h1>All is well. Here is the image in three sizes plus the original.</h1>' +
            '<div><img src="' + uploadfs.getUrl() + info.basePath + '.small.' + info.extension + '" /></div>' + 
            '<div><img src="' + uploadfs.getUrl() + info.basePath + '.medium.' + info.extension + '" /></div>' + 
            '<div><img src="' + uploadfs.getUrl() + info.basePath + '.large.' + info.extension + '" /></div>' +
            '<div><img src="' + uploadfs.getUrl() + info.basePath + '.' + info.extension + '" /></div>');       
        }
      });
    });

Note the use of `uploadfs.getUrl()` to determine the URL of the uploaded image. **Use this method consistently and your code will find the file in the right place regardless of the backend chosen.**

## Retrieving Information About Images

When you successfully copy an image into uploadfs with copyImageIn, the second argument to your callback has the following useful properties:

`width` (already rotated for the web if necessary, as with iPhone photos)

`height` (already rotated for the web if necessary, as with iPhone photos)

`originalWidth` (not rotated)

`originalHeight` (not rotated)

`extension` (`gif`,`jpg` or `png`)

You should record these properties in your own database if you need access to them later.

**When cropping, the uncropped size of the original image is not returned by uploadfs. It is assumed that if you are cropping you already know what the original dimensions were.**

The same information is available via `identifyLocalImage` if you want to examine a local file before handing it off to `copyImageIn`.

## Removing Files

Here's how to remove a file:

    uploadfs.remove('/profiles/me.jpg', function(e) { ... });

## Disabling Access To Files

This call shuts off web access to a file:

    uploadfs.disable('/profiles/me.jpg', function(e) { ... });

And this call restores it:

    uploadfs.enable('/profiles/me.jpg', function(e) { ... });

*Depending on the backend, `disable` may also block the copyOut method*, so be sure to call `enable` before attempting any further access to the file. (Unfortunately S3 does not offer an ACL that acts exactly like `chmod 000`, thus this slight inconsistency.)

## Configuration Options

Here's are the options I pass to `init()` in `sample.js`. Note that I define the image sizes I want the `copyImageIn` function to produce. No image will be wider or taller than the limits specified. The aspect ratio is always maintained, so one axis will often be smaller than the limits specified. Here's a hint: specify the width you really want, and the maximum height you can put up with. That way only obnoxiously tall images will get a smaller width, as a safeguard.

    {
      backend: 'local',
      uploadsPath: __dirname + '/public/uploads',
      uploadsUrl: 'http://localhost:3000' + uploadsLocalUrl,
      // Required if you use copyImageIn
      // Temporary files are made here and later automatically removed
      tempPath: __dirname + '/temp',
      imageSizes: [
        {
          name: 'small',
          width: 320,
          height: 320
        },
        {
          name: 'medium',
          width: 640,
          height: 640
        },
        {
          name: 'large',
          width: 1140,
          height: 1140
        }
      ],
      // Render up to 4 image sizes at once. Note this means 4 at once per call
      // to copyImageIn. There is currently no built-in throttling of multiple calls to
      // copyImageIn
      parallel: 4
    }

Here is an equivalent configuration for S3:

    {
      backend: 's3',
      // Get your credentials at aws.amazon.com
      secret: 'xxx',
      key: 'xxx',
      // You need to create your bucket first before using it here
      // Go to aws.amazon.com
      bucket: 'getyourownbucketplease',
      // I recommend creating your buckets in a region with 
      // read-after-write consistency (not us-standard)
      region: 'us-west-2',
      // Required if you use copyImageIn
      tempPath: __dirname + '/temp',
      imageSizes: [
        {
          name: 'small',
          width: 320,
          height: 320
        },
        {
          name: 'medium',
          width: 640,
          height: 640
        },
        {
          name: 'large',
          width: 1140,
          height: 1140
        }
      ],
      // Render up to 4 image sizes at once. Note this means 4 at once per call
      // to copyImageIn. There is currently no built-in throttling of multiple calls to
      // copyImageIn
      parallel: 4

    }

"Why don't you put the temporary files for imagemagick in S3?"

Two good reasons:

1. Imagemagick doesn't know how to write directly to S3.

2. Constantly copying things to and from S3 is very slow compared to working with local temporary files. S3 is only fast when it comes to delivering your finished files to end users. Resist the temptation to use it for many little reads and writes.

## Less Frequently Used Options

* In backends like imagemagick that support it, even the "original" is rotated for you if it is not oriented "top left," as with some iPhone photos. This is necessary for the original to be of any use on the web. But it does modify the original. So if you really don't want this, you can set the `orientOriginals` option to `false`.

* It is possible to pass your own custom storage module instead of `local` or `s3`. Follow `local.js` or `s3.js` as a model, and specify your backend like this:

    storage: require('mystorage.js')

* You may specify an alternate image processing backend via the `image` option. Two backends, `imagemagick` and `sip`, are built in. `sip` is available only on Macs (it is a standard MacOS utility, available out of the box). `sip` is fast but it can't auto-orient rotated iPhone photos for you. You may also supply an object instead of a string to use your own image processor.

## Important Concerns With S3

**Be aware that uploads to Amazon S3's us-standard region are not guaranteed to be readable the moment you finish uploading them.** This is a big difference from how a regular filesystem behaves. One browser might see them right away while another does not. This is called "eventual consistency." To avoid this, you can use an alternate S3 region such as `us-west-2` (Oregon). However, also be aware that updates of an existing file or deletions of a file still won't be instantly seen everywhere, even if you don't use the `us-standard` region. To avoid this problem, include a version number or randomly generated ID in each filename.

In `sample.js` I configure Express to actually serve the uploaded files when using the local backend. When using the s3 backend, you don't need to do this, because your files are served from S3. S3 URLs look like this:

    http://yourbucketname.s3.amazonaws.com/your/path/to/something.jpg

But your code doesn't need to worry about that. If you use `uploadfs.getUrl()` consistently, code written with one backend will migrate easily to the other.

It's up to you to create an Amazon S3 bucket and obtain your secret and key. See sample.js for details.

S3 support is based on the excellent [knox](https://npmjs.org/package/knox) module.

## About P'unk Avenue and Apostrophe

`uploadfs` was created at [P'unk Avenue](http://punkave.com) for use in many projects built with Apostrophe, an open-source content management system built on node.js. Appy isn't mandatory for Apostrophe and vice versa, but they play very well together. If you like `uploadfs` you should definitely [check out apostrophenow.org](http://apostrophenow.org). Also be sure to visit us on [github](http://github.com/punkave).

## Support

Feel free to open issues on [github](http://github.com/punkave/uploadfs).

<a href="http://punkave.com/"><img src="https://raw.github.com/punkave/uploadfs/master/logos/logo-box-builtby.png" /></a>

## Changelog

### CHANGES IN 1.1.0

The new `disable` and `enable` methods turn web access to the specified path off and on again, respectively. The new `getImageSizes` method simply gives you access to the image sizes that are currently configured.

There are no changes elsewhere in the code.

### CHANGES IN 1.0.0

None! Since the additions in version 0.3.14 we've had no real problems. We now support both alternate storage backends and alternate image rendering backends. Test coverage is thorough and everything's passing. What more could you want? It's time to declare it stable.

### CHANGES IN 0.3.15

Decided that imagecrunch should output JSON, so that's now what the backend expects.

### CHANGES IN 0.3.14

In addition to storage backends, you may also supply alternate image processing backends. The `backend` option has been renamed to `storage`, however `backend` is accepted for backwards compatibility. The `image` option has been introduced for specifying an image processing backend. In addition to the existing `imagemagick` backend, there is now an `imagecrunch` backend based on the Mac-specific [imagecrunch](http://github.com/punkave/imagecrunch) utility.

If you do not specify an `image` backend, uploadfs will look for imagecrunch and imagemagick in your PATH, stopping as soon as it finds either the `imagecrunch` command or the `identify` command.

### CHANGES IN 0.3.13

`copyImageIn` has been rewritten to run more than 4x faster! We now generate our own imagemagick `convert` pipeline which takes advantage of two big optimizations:

* Load, orient and crop the original image only once, then output it at several sizes in the same pipeline. This yields a 2x speedup.
* First scale the image to the largest size desired, then scale to smaller sizes based on that as part of the same pipeline, without creating any lossy intermediate files. This yields another 2x speedup and a helvetica of designers were unable to see any difference in quality. ("Helvetica" is the collective noun for a group of designers.)

The new `parallel` option allows you to specify the maximum number of image sizes to render simultaneously. This defaults to 1, to avoid using a lot of memory and CPU, but if you are under the gun to render a lot of images in a hurry, you can set this as high as the number of image sizes you have. Currently there is no throttling mechanism for multiple unrelated calls to `uploadfs.copyImageIn`, this option relates to the rendering of the various sizes for a single call.

### CHANGES IN 0.3.11

The new `parallel` option allows you to specify the maximum number of image sizes to render simultaneously. This defaults to 1, to avoid using a lot of memory and CPU, but if you are under the gun to render a lot of images in a hurry, you can set this as high as the number of image sizes you have. Currently there is no throttling mechanism for multiple unrelated calls to `uploadfs.copyImageIn`, this option relates to the rendering of the various sizes for a single call.

### CHANGES IN 0.3.7-0.3.10

Just packaging and documentation. Now a P'unk Avenue project.

### CHANGES IN 0.3.6

The `uploadfs` functionality for identifying a local image file via ImageMagick has been refactored and made available as the `identifyLocalImage` method. This method is primarily used internally but is occasionally helpful in migration situations (e.g. "I forgot to save the metadata for any of my images before").

### CHANGES IN 0.3.5

Starting in version 0.3.5, you can set the quality level for scaled JPEGs via the scaledJpegQuality option, which defaults to 80. You can pass this option either when initializing `uploadfs` or on individual calls to `copyImageIn`. This option applies only to scaled versions of the image. If uploadfs modifies the "original" image to scale or orient it, Imagemagick's default behavior stays in effect, which is to attempt to maintain the same quality level as the original file. That makes sense for images that will be the basis for further cropping and scaling but results in impractically large files for web deployment of scaled images. Thus the new option and the new default behavior.

### CHANGES IN 0.3.4

Starting in version 0.3.4, the getTempPath() method is available. This returns the same `tempPath` that was supplied to uploadfs at initialization time. Note that at this point the folder is guaranteed to exist. This is useful when you need a good place to `copyOut` something to, for instance in preparation to `copyImageIn` once more to carry out a cropping operation.

### CHANGES IN 0.3.3

Starting in version 0.3.3, cropping is available. Pass an options object as the third parameter to `copyImageIn`. Set the `crop` property to an object with `top`, `left`, `width` and `height` properties, all specified in pixels. These coordinates are relative to the original image. **When you specify the `crop` property, both the "full size" image copied into uploadfs and any scaled images are cropped.** The uncropped original is NOT copied into uploadfs. If you want the uncropped original, be sure to copy it in separately. The `width` and `height` properties of the `info` object passed to your callback will be the cropped dimensions.

Also starting in version 0.3.3, `uploadfs` uses the `gm` module rather than the `node-imagemagick` module for image manipulation, but configures `gm` to use imagemagick. This change was made because `node-imagemagick` has been abandoned and `gm` is being actively maintained. This change has not affected the `uploadfs` API in any way. Isn't separation of concerns wonderful?

### CHANGES IN 0.3.2

Starting in version 0.3.2, you can copy files back out of uploadfs with `copyOut`. You should not rely heavily on this method, but it is occasionally unavoidable, for instance if you need to crop an image differently. When possible, cache files locally if you may need them locally soon.

### CHANGES IN 0.3.0

Starting in version 0.3.0, you must explicitly create an instance of uploadfs. This allows you to have more than one, separately configured instance, and it also avoids serious issues with modules not seeing the same instance automatically as they might expect. For more information see [Singletons in #node.js modules cannot be trusted, or why you can't just do var foo = require('baz').init()](http://justjs.com/posts/singletons-in-node-js-modules-cannot-be-trusted-or-why-you-can-t-just-do-var-foo-require-baz-init).

Existing code that isn't concerned with sharing uploadfs between multiple modules will only need a two line change to be fully compatible:

    // CHANGE THIS
    var uploadfs = require('uploadfs');

    // TO THIS (note the extra parens)
    var uploadfs = require('uploadfs')();

If you use uploadfs in multiple source code files, you'll need to pass your `uploadfs` object explicitly, much as you pass your Express `app` object when you want to add routes to it via another file.


