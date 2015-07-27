---
title: "How I Converted a Large mod_perl Project to a PSGI App"
layout: post
category: Perl
---

## Background

I've been working on a large intranet application for several years. The user
facing portion of the application is a set of CGI scripts using `CGI.pm`. I have
everything running under Apache mod\_perl 2.0. The pages are handled with
[ModPerl::Registry::Prefork](http://search.cpan.org/dist/mod_perl/docs/api/ModPerl/RegistryPrefork.pod),
which allows normal CGI scripts to run under mod\_perl.

(Note: since `ModPerl::Registry::Prefork` wraps your script in an outer `sub
{}` that can be executed by the handler, you have to avoid file-scoped lexicals
for sharing variables within your script. The easy fix is to change file-scoped
`my` variables to `our` variables. All of my scripts had this in place already,
which made the transaction to PSGI easier.)

## Drawbacks

This environment works fine, but it has a few drawbacks in practice:

* The application is tied to Apache, which means I have to configure a full
  Apache stack for development.
* It's difficult to use mod\_perl with tools like
  [plenv](https://github.com/tokuhirom/plenv) and
  [Carton](https://github.com/perl-carton/carton), which allow an
  application-specific Perl installation. So I end up polluting the system
  Perl.
* Acceptance testing is difficult, since I can't run the application pages
  without an Apache stack.

## PSGI/Plack

Enter [PSGI/Plack](http://plackperl.org/). This is a modern specfication and
set of supporting modules inspired by Ruby's Rack. Think of it as a standard
way of defining applications and servers so they can work together in multiple
configurations.

What it meant for me was that if I could change my application from working
with mod\_perl-specific specification (`ModPerl::Registry::Prefork`) to working
with the PSGI specification, I would no longer have to run the application
under Apache, and I would be able to address all of the drabacks mentioned
above.

## Goals

My goals for this change were:

* I would like to move away from `CGI.pm`, but don't want to have to do it right
  now. Ideally, I would like to not have to change any of the current CGI
  scripts.
* I don't want to have to rearrange the file structure of my application right
  away.  The current application has executable CGI scripts, template files,
  stylesheets, javascripts, and images all under the same tree.

Both of these would allow me to get to PSGI *now*, without having to make
changes to all of the scripts. Once I am on PSGI, I can work on restructuring
the app to get away from `CGI.pm` and use a more intelligent file layout.

## What's in the Box?

The toughest part of this whole thing was trying to wrap my head around
what all this stuff in Plack/PSGI *was*, exactly. There are a ton of tools
included, and it took some time to figure out what I did and did not need
for my task.

The most important concept to understand is a PSGI application. A PSGI
application is a code reference that is executed to handle an HTTP request and
return an HTTP response. The format of the request and response are defined by
the PSGI specification.

There are essentially four types of tools in the Plack toolbox (or in
separately available modules):

* Servers and server adapters. These are either standalone web servers or
  adapters for popular web servers that enable them to run PSGI applications. I
  didn't need to worry about this yet; once I had a PSGI application, I should
  be able to run it on any of these servers using these tools.
* Frameworks. A framework is a "canned" PSGI application that you can hook your
  functionality into. I originally thought that I needed to pick one of these
  frameworks, but that turned out to be incorrect. More on that below.
* Middleware. Plack middleware are code references that can be injected into
  the request/response cycle to do all kinds of things, including modifying the
  request before it gets to the application or modifying the response after it
  is returned by the application. Middleware is extremely powerful and easy
  to use, and there are tons of pre-built middleware modules.
* Applications. Plack comes with a set of pre-built applications that are ready
  to use right away; you don't have to add any code. Each of these has a
  specific single function. For example, there is one to serve static files
  from a directory tree. There is another to execute CGI scripts in a cgi-bin
  directory.

## Getting to a Solution

When I first started looking at all of this, nothing seemed to be the right
answer. I didn't want a Framwork, because I would have to rewrite all my
scripts to adapt to the Framework's calling conventions. The
`Plack::APP::CGIBin` application seemed promising. From the description:

> Plack::App::CGIBin allows you to load CGI scripts from a directory and
> convert them into a PSGI application.

OK, this is close to what I'm doing, but the problem is that my directory
contains more than just CGI scripts. It also contains static assets that
need to be served (stylesheets, etc.) as well as other files that should *not*
be served (templates). `Plack::App::CGIBin` will just blindly treat all
the files in my directory as CGI scripts to be executed.

The solution finally came to me when I examined the `Plack::App::CGIBin` source
code. It turns out that `Plack::App::CGIBin` just subclasses
`Plack::App::File`, which serves static files from a directory.
`Plack::App::File` takes care of converting a URL to a file path, responding
with a 404 for non-existent files, and then returning the file contents.
`Plack::App::CGIBin` replaces the file sending with code to execute the file as
a script.

So what I needed was my own custom version of `Plack::App::File` that would
do the following things:

* Map urls to file names from a document root. `Plack::App::File` does that out
  of the box.
* Send a 404 response on a non-existent file. Again, already does that.
* Send stylesheets, images, etc. as static content. Already does that as well.
* Send a 404 response on my templates or other files that should not be
  served to the client. New functionality.
* Handle my CGI scripts by executing them. New functionality I can steal from
  `Plack::App::CGIBin`
* Treat `index.pl` as the script to run if a directory url is requested
  (similar to Apache's `DirectoryIndex` directive). I will use middleware for
  this.

## The Final Application

Here is what I finally came up with. I named the file `app.psgi`, which is the
default application file name. This means I can start a local server for this
application just by running the command `plackup`. Sweet.

I placed the `app.psgi` file in the root directory of my application. All the
content that I want to serve is in the `www` subdirectory below this root.

A line-by-line analysis of the application follows.

```perl
     1	#!/usr/bin/env perl
     2	
     3	package MyApp;
     4	
     5	use common::sense;
     6	
     7	use File::Basename;
     8	use File::Spec;
     9	use Plack::App::WrapCGI;
    10	use SUPER;
    11	
    12	use parent 'Plack::App::File';
    13	
    14	# only files ending with these suffixes will be served statically
    15	my @STATIC_SUFFIXES = qw(.css .gif .ico .js .png);
    16	
    17	sub should_serve_static
    18	{
    19	  my ($self, $suffix) = @_;
    20	  $self->{static_suffixes} ||= do {
    21	    my %h;
    22	    @h{@STATIC_SUFFIXES} = ();
    23	    \%h;
    24	  };
    25	  exists($self->{static_suffixes}{$suffix});
    26	}
    27	
    28	sub serve_path
    29	{
    30	  my ($self, $env, $file) = @_;
    31	
    32	  # get path and suffix
    33	  my ($name, $path, $suffix) = fileparse($file, qr/\.[^.]*/);
    34	
    35	  # serve as a perl script (adapted from Plack::App::CGIBin serve_path method)
    36	  if ($suffix eq '.pl') {
    37	    local @{$env}{qw(SCRIPT_NAME PATH_INFO)} = @{$env}{qw( plack.file.SCRIPT_NAME plack.file.PATH_INFO )};
    38	    local $env->{DOCUMENT_ROOT} = $self->root;
    39	    local $env->{GATEWAY_INTERFACE} = 'PSGI/1.1';
    40	    local $env->{REMOTE_USER} = 'bshowalt';
    41	    my $app = $self->{_compiled}{$file} ||= Plack::App::WrapCGI->new(script => $file)->to_app;
    42	    return $app->($env);
    43	  }
    44	
    45	  # serve as static file
    46	  return super if $self->should_serve_static($suffix);
    47	
    48	  # otherwise not found
    49	  $self->return_404;
    50	}
    51	
    52	package main;
    53	
    54	use common::sense;
    55	
    56	use FindBin;
    57	use Plack::Builder;
    58	
    59	builder {
    60	  enable 'DirIndex', dir_index => 'index.pl';
    61	  MyApp->new(
    62	    root => "$FindBin::Bin/www"
    63	  )->to_app;
    64	};
```

* Line 1: Perl scripts should be loaded with `#!/usr/bin/env perl` instead of
  `#!/usr/bin/perl`, so tools like `rbenv` can use a local Perl environment.
* Line 3-13: I'm creating my custom application class, based on
  `Plack::App::File`.
* Lines 14-26. Since I have some files I want to serve statically (e.g.
  stylesheets), and some that I don't (e.g. templates), I created a whitelist
  of suffixes to be served statically. The `should_serve_static` method will
  return true of a given suffix represents a file that should be served
  statically.
* Line 28: `serve_path` is overriding `Plack::App::File`'s method. Here
  is the meat of my application.
* Line 30: `serve_path` receives the PSGI evironment and the full path
  of the file that needs to be served. (This file exists; a 404 would
  already have been returned if a non-existent file was requested).
* Line 33: Split the file path into separate components. The suffix will
  be the key to how the file will be handled.
* Line 36-43: If the suffix is '.pl', the file needs to be executed as a CGI
  script. The code here is lifted from `Plack::App::CGIBin`. Lines 38-40 set
  some environment variables that are needed by my particular application.
  Note that line 41 creates an app (a coderef), and line 42 executes the
  app, returning whatever the app returns (which in this case, is the output
  of the CGI program, in the PSGI response format).
* Line 46: If the file was not a .pl, it should be served statically, but only
  if it is one of the whitelisted suffixes. The call to `super` invokes
  `Plack::App::File`'s handling, which will return the file contents.
* Line 49: If we get here, the suffix was not .pl and was not one of the
  whitelisted static suffixes, so it must be a template or some other file I do
  not wish to be served. We will treat it as a "Not Found" response. The
  `return_404` method is defined in `Plack::App::File`.
* Line 52: Now that I have my custom application class, we can start
  the main application. The end result needs to be a PSGI application,
  which is a coderef.
* Line 57: `Plack::Builder` is a DSL that lets you build an app by
  composing other apps with middleware.
* Line 59: The call to `builder` wraps the DSL and returns an app coderef.
* Line 60: This line enables the `Plack::Middleware::DirIndex` middleware,
  which changes the request for a directory to a request for a specific file
  within the directory (`index.pl` in this case). This is handling what
  Apache's `DirectoryIndex` directive was doing for me. Note that I did not
  need to add anything to my application to handle this case; the middleware
  takes care of it, and my application will only see the `index.pl` file.
  (Also, if a directory does not have an `index.pl` file, the middleware will
  not adjust the request, and `Plack::App::File` will treat the request as a
  404.
* Line 61-63: This constructs my application, setting the root directory to the
  `www` directory below the `app.psgi` file. The `to_app` method returns the
  psgi application.

There are some features I still need to handle, most of which will probably
be done through middleware:

* Authentication and authorization. This is being done now through some
  Apache directives in .htaccess files.
* Error response pages (the standard pages are very minimalistic)

This conversion turned out really well. The application runs great, and I did
not have to touch any of the actual CGI scripts or move any of the files
around. I am now able to run and test the application outside of the Apache
environment, and I'm ready to start migrating it away from `CGI.pm` and into a
new framework, which might be an existing one or one that I create.

