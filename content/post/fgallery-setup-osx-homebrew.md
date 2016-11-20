+++
date = "2016-11-19T13:06:35+01:00"
tags = ["tech", "servers", "osx", "blog", "linux", "perl", "howto", "photography"]
title = "Setting up fgallery on OS X with Homebrew"
topics = ["howto", "tech", "osx", "photography", "osx"]

+++

### How'd I Pick `fgallery` Anyway?

Currently this site is being generated offline as a series of static images, HTML, CSS, and JS files that get served almost entirely out of cache through the CDN provided by [Cloudflare](https://www.cloudflare.com/).  This is made possible by a piece of software called [Hugo](https://gohugo.io/).  Hugo takes a series of Markdown formatted text files, some HTML/CSS/JS templates, and a theme made of HTML/CSS/JS and generates this entire site each time I run the command `hugo` inside my site repository.  This confers a lot of advantages over a more traditional approach such as using a CMS like Wordpress, such as:

* Every single thing on my site is cachable by the end-user's browser and the CDN in-between
* Nothing on the site relies on my server's power, so I can use the cheapest instance available to host the site
* Greatly reduces the bandwidth required of my server to host the site
* Trade's off the client doing a bit of work during rendering to support any device nicely
* Gives me a progressive responsive site on all device types and connection speeds (which I have relied on first-hand in my travels)
* Since all components of the site are served to the client, it works better for accessibility
* Since all components of the site are contained in the repository, I don't need to maintain a database server or database backups to restore the site after an issue.

Now enter hosting photos.  Photos are a bit of a different animal to a bunch of text.  As you've probably noticed, this site mostly uses Flickr or Instagram to host photos and embed them, without relying on self-hosting.  This is because photos are inherently bandwidth intensive, difficult to organize, and difficult to make accessible to multiple device types and connection speeds. There's a lot of great solutions out there which rely on server-side processing to dynamically generate a photo gallery, and these are very popular.  They're similar in approach to the method of using Wordpress or a similar CMS as a blogging platform.  But again, I wanted to be able to rely heavily on a CDN to ensure that users anywhere in the world get performant access to my site without needing multiple server instances, and I don't want the hassle of dealing with database servers and backups.  So, just like the text of this site I needed a static site generator for photo galleries.

It turns out there's not many options, unlike for static site generators for blogs.  But, luckily, there is what appears to be a very good option.  Enter [`fgallery`](https://www.thregr.org/~wavexx/software/fgallery/).  It's minimalist in design, uses JS to do progressive loading of images and dynamic scaling to support all screen sizes and device types, and since it output static content not only can I easily integrate it with the existing site it is possible to cache 100% of the output on the CDN.

There's just one catch... `fgallery` is written in Perl using threads and OpenCV and therefore relies on a handful of dependencies that are painful to install on OS X, my primary personal/development platform.  Here's the process I followed to get this all working.  

##### **_NOTE: This is going to take awhile!_**

### Use `homebrew` When You Can

We need to start off by installing the dependencies you can with Homebrew, although many of them are going to be a bit more of a hassle.

```
brew install imagemagick lcms2 pngcrush jpeg jpegoptim p7zip exiftool fbida libexif python openexr eigen tbb
```

Once this completes, we get to move onto the more annoying part.

### Use `perlbrew` When You Have To

Install `perlbrew` by the unfortunately terribly insecure process of piping shell scripts to `bash`.  I recommend you grab the script, read it, then manually run it, but for expediency I am going to provide the commands as the documentation provides them.

```
curl -L https://install.perlbrew.pl | bash
```

Once this completes, set up your shell environment and start a new shell session or run `perlbrew init`  We need to install Perl with threading support as a requirement for `fgallery`

```
perlbrew install --as perl-5.25.6t -Dusethreads perl-5.25.6
perlbrew switch perl-5.25.6t
```

The install process takes some time as it compiles `perl` from source.  When it's completed you should have a self-contained working version of Perl 5 that's separated from your system Perl environment.

Next up, we need to install `cpanm` and then install our Perl module dependencies for `fgallery`

```
curl -L http://cpanmin.us | perl - App::cpanminus
```

Thanks to `cpanm` you've already got a minimal CPAN environment bootstrapped, so you can install modules with the `cpanm` command and skip the annoyingly verbose CPAN console.

Install `Image::ExifTool` and `Cpanel::JSON::XS`

```
cpanm Image::ExifTool
cpanm Cpanel::JSON::XS
```

### Install `facedetect` and its Remaining Dependencies

`facedetect` is an optional dependency that provides some nice features by the same author as `fgallery`.  It's written in Python and it relies on OpenCV to do image detection.

First we need to get some dependencies installed.

```
brew tap homebrew/science
brew tap homebrew/python
brew install numpy
brew install tesseract --with-opencl
brew install opencv3 --with-contrib --with-tbb
```

If you run into issues installing `opencv3` above, it's most likely related to the fact Apple deprecated QTKit in Xcode 8/Sierra in favor of AVFoundation without really giving folks much/any warning.  There is a fix in the HEAD of OpenCV already, the Homebrew recipe supports hooking Quicktime instead, or you can edit the recipe to disable video support since it's not needed for our purposes by adding `-DBUILD_opencv_videoio=OFF` as a cmake flag.

I'm choosing to use HEAD, so I modified things to be instead: `brew install opencv3 --HEAD --with-contrib --with-tbb`

Then we symlink the `cv2.so` file to your Python site-packages directory with
```
ln -s /usr/local/opt/opencv3/lib/python2.7/site-packages/cv2.so /usr/local/lib/python2.7/site-packages/
```

Finally, we need to copy the OpenCV data to the correct place.  Clone the [OpenCV repository](https://github.com/opencv/opencv) and change directory into it before running the following.

```
mkdir -pv /usr/local/share/opencv/
cp -r data/ /usr/local/share/opencv
```

Once these prerequesites are installed you just need to clone the [git repository for `facedetect`](https://github.com/wavexx/facedetect) and follow the install instructions.

You may encounter issues with `facedetect` expecting the OpenCV data in `/usr/share/` instead of `/usr/local/share/`.  You can fix this by manually modifying the software to look in the correct path.

`DATA_DIR = '/usr/share/opencv/'` on line 37 should instead say `DATA_DIR = '/usr/local/share/opencv/'`

### Install `fgallery`

To install `fgallery` clone the [git repository](https://github.com/wavexx/fgallery) and then run the following commands.

```
cp -r fgallery/ /usr/local/share/fgallery
ln -s /usr/local/share/fgallery/fgallery /usr/local/bin
```

If everything has been installed correctly you should now be able to just run `fgallery` at your shell and get back a help message.


### Integrating `fgallery` with `hugo`

These steps are optional.  This is just what I did here.  I added a sidebar item via `config.toml` for `/gallery/` and copied the output of `fgallery` into the content path as `/gallery/`.  It's that simple.  No need to inject front matter.  Next time Hugo runs, yourdomain.com/gallery/ is the photo gallery.

### Conclusion

It takes awhile to compile all the necessary software from scratch on OS X, but once it's all said and done `fgallery` is a really nice piece of software that seems to do what I need.  I probably need to do more work on stylesheets to make it fit more in tune with my existing Hugo theme.  Maybe add the sidebar for navigation back.  All these things are going to require either hacking on fgallery or writing a helper script to handle the "integration".

That'll all be for another time and maybe another article.
