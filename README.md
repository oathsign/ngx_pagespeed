# ngx_pagespeed

This is the [nginx](http://nginx.org/) port of
[mod_pagespeed](https://developers.google.com/speed/pagespeed/mod).

**ngx_pagespeed is alpha**. If you are interested in test-driving the module, or
contributing to the project, see below. For feedback, questions, and to follow
the progress of the project:

- [ngx-pagespeed-discuss Google Group](https://groups.google.com/forum/#!forum/ngx-pagespeed-discuss)

---

The goal of ngx_pagespeed is to speed up your site and reduce page load time by
automatically applying web performance best practices to pages and associated
assets (CSS, JavaScript, images) without requiring that you modify your existing
content or workflow. Features include:

- Image optimization: stripping meta-data, dynamic resizing, recompression
- CSS & JavaScript minification, concatenation, inlining, and outlining
- Small resource inlining
- Deferring image and JavaScript loading
- HTML rewriting
- Cache lifetime extension
- and [more](https://developers.google.com/speed/docs/mod_pagespeed/config_filters)
  - Note: not all mod_pagespeed features work in ngx_pagespeed yet.

To see ngx_pagespeed in action, with example pages for each of the
optimizations, see our <a href="http://ngxpagespeed.com">demonstration site</a>.

## How to build

Because nginx does not support dynamic loading of modules, you need to compile
nginx from source to add ngx_pagespeed.

1. Install dependencies:

   ```bash
   # These are for RedHat, CentOS, and Fedora.
   $ sudo yum install git gcc-c++ pcre-dev pcre-devel zlib-devel make

   # These are for Debian. Ubuntu will be similar.
   $ sudo apt-get install git-core build-essential zlib1g-dev libpcre3 libpcre3-dev
   ```

2. Check out ngx_pagespeed:

   ```bash
   $ cd ~
   $ git clone https://github.com/pagespeed/ngx_pagespeed.git
   ```

3. Download and build nginx:

   ```bash
   $ # check http://nginx.org/en/download.html for the latest version
   $ wget http://nginx.org/download/nginx-1.3.15.tar.gz
   $ tar -xvzf nginx-1.3.15.tar.gz
   $ cd nginx-1.3.15/
   $ ./configure --add-module=$HOME/ngx_pagespeed
   $ make install
   ```

If `make` fails with `unknown type name ‘off64_t’`,
add `--with-cc-opt='-DLINUX=2 -D_REENTRANT -D_LARGEFILE64_SOURCE -march=i686 -pthread'`
to `./configure` and try to `make` again.

If `configure` fails with `checking for psol ... not found` then open
`objs/autoconf.err` and search for `psol`.

If it's not clear what's wrong from
the error message, then send it to the [mailing
list](https://groups.google.com/forum/#!forum/ngx-pagespeed-discuss) and we'll
have a look at it.

This will use a binary distribution of the PageSpeed Optimization Library
(PSOL).  If you would like to build PSOL from source there's [a more involved
process](https://github.com/pagespeed/ngx_pagespeed/wiki/Building-PSOL-From-Source).

If you're using Tengine you can [install ngx_pagespeed without
recompiling Tengine](https://github.com/pagespeed/ngx_pagespeed/wiki/Using-ngx_pagespeed-with-Tengine).

## How to use

In your `nginx.conf`, add to the main or server block:

    pagespeed on;

    # needs to exist and be writable by nginx
    pagespeed FileCachePath /var/ngx_pagespeed_cache;

In every server block where pagespeed is enabled add:

    #  Ensure requests for pagespeed optimized resources go to the pagespeed
    #  handler and no extraneous headers get set.
    location ~ "\.pagespeed\.([a-z]\.)?[a-z]{2}\.[^.]{10}\.[^.]+" {
      add_header "" "";
    }
    location ~ "^/ngx_pagespeed_static/" { }
    location ~ "^/ngx_pagespeed_beacon$" { }
    location /ngx_pagespeed_statistics { allow 127.0.0.1; deny all; }
    location /ngx_pagespeed_message { allow 127.0.0.1; deny all; }

To confirm that the module is loaded, fetch a page and check that you see the
`X-Page-Speed` header:

```bash
$ curl -s -D- 'http://localhost:8050/some_page/' | grep X-Page-Speed
X-Page-Speed: 1.4.0.0-2729
```

Looking at the source of a few pages you should see various changes, such as
urls being replaced with new ones like `yellow.css.pagespeed.ce.lzJ8VcVi1l.css`.

Most mod_pagespeed configuration directives work in ngx_pagespeed after a small
adjustment: replace '"ModPagespeed"' with '"pagespeed "':

    mod_pagespeed.conf:
      ModPagespeedEnableFilters collapse_whitespace,add_instrumentation
      ModPagespeedRunExperiment on
      ModPagespeedExperimentSpec id=3;percent=50;default
      ModPagespeedExperimentSpec id=4;percent=50

    ngx_pagespeed.conf:
      pagespeed EnableFilters collapse_whitespace,add_instrumentation;
      pagespeed RunExperiment on;
      pagespeed ExperimentSpec "id=3;percent=50;default";
      pagespeed ExperimentSpec "id=4;percent=50";

For more configuration details, see the [differences from mod_pagespeed
configuration](https://github.com/pagespeed/ngx_pagespeed/wiki/Configuration-differences-from-mod_pagespeed)
wiki page.

## Testing

The generic Pagespeed system test is ported, and all but three tests pass.  To
run it you need to first build nginx.  You also need to check out mod_pagespeed,
but we can take a shortcut and do this the easy way, without gyp, because we
don't need any dependencies:

```bash
$ svn checkout https://modpagespeed.googlecode.com/svn/trunk/ mod_pagespeed
```

Then run:

```bash
test/run_tests.sh \
  primary_port \
  secondary_port \
  mod_pagespeed_dir \
  nginx_executable_path
```

For example:

```bash
$ test/run_tests.sh 8050 8051 /path/to/mod_pagespeed \
    /path/to/sbin/nginx
```

All of these paths need to be absolute.

This should print out a lot of lines like:

    TEST: Make sure 404s aren't rewritten
          check_not fgrep /mod_pagespeed_beacon /dev/fd/63

and then eventually:

    Failing Tests:
      In-place resource optimization
      In-place resource optimization
      In-place resource optimization
      convert_meta_tags
      insert_dns_prefetch
      insert_dns_prefetch
    FAIL.
    With serf fetcher setup.

Each of these failed tests is a known issue:
 - [In-place resource
   optimization](https://github.com/pagespeed/ngx_pagespeed/issues/42)
 - [compression is enabled for rewritten JS.](
    https://github.com/pagespeed/ngx_pagespeed/issues/70)
   - If you're running a version of nginx without etag support (pre-1.3.3) you
     won't see this issue, which is fine.
 - [convert_meta_tags](https://github.com/pagespeed/ngx_pagespeed/issues/56)
 - [insert_dns_prefetch](https://github.com/pagespeed/ngx_pagespeed/issues/114)

If it fails with:

    TEST: PHP is enabled.
    ...
    in 'PHP is enabled.'
    FAIL.

the problem is that the test expects a php server to be running on port 9000:

```bash
$ sudo apt-get install php5-cgi
$ php-cgi -b 127.0.0.1:9000 &
```

If it fails with some other error, that's a problem, and it would be helpful for
you to [submit a bug](https://github.com/pagespeed/ngx_pagespeed/issues/new).

Log files are in `test/tmp/error.log` and `test/tmp/access.log`.

### Testing with memcached

Start an memcached server:

```bash
$ memcached -p 11211
```

In `ngx_pagespeed/test/pagespeed_test.conf.template` uncomment:

    pagespeed MemcachedServers "localhost:11211";
    pagespeed MemcachedThreads 1;

Then run the system test as above.

### Testing with valgrind

If you set the environment variable `USE_VALGRIND=true` then the tests will run
with valgrind:

```bash
USE_VALGRIND=true test/nginx_system_test.sh ...
```