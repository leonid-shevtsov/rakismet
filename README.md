Rakismet
========

**Akismet** (<http://akismet.com/>) is a collaborative spam filtering service.
**Rakismet** is easy Akismet integration with Rails and rack apps. TypePad's
AntiSpam service and generic Akismet endpoints are supported.

Getting Started
===============

Once you've installed the Rakismet gem and added it to your application's Gemfile,
you'll need an API key from the folks at WordPress. Head on over to
http://wordpress.com/api-keys/ and sign up for a new username.

Configure the Rakismet key and the URL of your application by setting the following
in an initializer or application.rb:

    config.rakismet.key = 'your wordpress key'
    config.rakismet.url = 'http://yourdomain.com/'

If you wish to use another Akismet-compatible API provider such as TypePad's
antispam service, you'll also need to set `config.rakismet.host` to your service
provider's endpoint.

Adding Rakismet to Your Application
-----------------------------------

First, introduce Rakismet to your model:

    class Comment
      include Rakismet::Model
    end

Rakismet sends the following information to the spam-hungry robots at Akismet:

    author        : name submitted with the comment
    author_url    : URL submitted with the comment
    author_email  : email submitted with the comment
    comment_type  : Defaults to comment but you can set it to trackback, pingback, or something more appropriate
    content       : the content submitted
    permalink     : the permanent URL for the entry the comment belongs to
    user_ip       : IP address used to submit this comment
    user_agent    : user agent string
    referrer      : referring URL (note the spelling)

By default, Rakismet just looks for attributes or methods on your class that
match these names. You don't have to have accessors that match these exactly,
however. If yours differ, just tell Rakismet what to call them:

    class Comment
      include Rakismet::Model
      attr_accessor :commenter_name, :commenter_email
      rakismet_attrs :author => :commenter_name,
                     :author_email => :commenter_email
    end

Or you can pass in a proc, to access associations:

    class Comment < ActiveRecord::Base
      include Rakismet::Model
      belongs_to :author
      rakismet_attrs :author => proc { author.name },
                     :author_email => proc { author.email }
    end

Checking For Spam
-----------------

Rakismet provides three methods for interacting with Akismet:

 * `spam?`

Simply call `@comment.spam?` to get a true/false response. True means it's spam,
false means it's not. (In case of an error, `@comment.spam?` will also return
false. If you want to make sure your Akismet requests are behaving properly,
you can check `@comment.akismet_response`. Anything other than "true" or
"false" means you got an error. But as long as you're collecting the data
above, it's probably safe to rely on `@comment.spam?` alone.)

 * `ham!` and 
 * `spam!`

Akismet works best with your feedback. If you spot a comment that was
erroneously marked as spam, `@comment.ham!` will resubmit to Akismet, marked
as a false positive. Likewise if they missed a spammy comment,
`@comment.spam!` will resubmit marked as spam.

Optional Request Variables
--------------------------

Akismet wants certain information about the request environment: remote IP, the
user agent string, and the HTTP referer when available. Normally, Rakismet
asks your model for these. Storing this information on your model allows you to
call the `spam?` method at a later time, e.g. your comments are in an
administrative queue or you're using a background job to process them.

You don't need to have these three attributes on your model, however. If you
choose to omit them, Rakismet will instead look at the current request (if one
exists) and ask it for the values instead.

This means that if you are **not storing request variables**, you must call
`spam?` from within the controller action that handles comment submissions. That
way the IP, user agent, and referer will belong to the person submitting the
comment. If you were to call `spam?` at a later time, the request information would
be missing or invalid. 

If you've decided to handle the request variables yourself and would like to
disable the middleware responsible for inspecting each request, add this to your
app initialization:

    config.rakismet.use_middleware = false

FAQ
===

Akismet thinks all of my test data is spam!
-------------------------------------------

Akismet needs enough information to decide if your test data is spam or not.
Try to supply as much as possible, especially the author name and request
variables.

How can I simulate a spam submission?
-------------------------------------

Most people have the opposite problem, where Akismet doesn't think anything is
spam. The only guaranteed way to trigger a positive spam response is to set the
comment author to "viagra-test-123".

If you've done this and `spam?` is still returning false, you're probably
missing the user IP or one of the key/url config variables. One way to check is
to call `@comment.akismet_response`. If you are missing a required field or
something else went wrong, this will hold the error message returned by
Akismet. If your comment was processed normally, this value will simply be
"true" or "false".

Can I use Rakismet with a different ORM or framework?
-----------------------------------------------------

Sure. Rakismet doesn't care what your persistence layer is. It will work with
Datamapper, a NoSQL store, or whatever next month's DB flavor is.

Rakismet also has no dependencies on Rails or any of its components, and only uses
a small Rack middleware object to do some of its magic. Depending on your
framework, you may have to modify this slightly and/or manually place it in your
stack.

You'll also need to set a few config variables by hand. Instead of
`config.rakismet.key`, `config.rakismet.url`, and `config.rakismet.host`, set
these values directly with `Rakismet.key`, `Rakismet.url`, and `Rakismet.host`.

---------------------------------------------------------------------------

If you have any implementation or usage questions, don't hesitate to get in
touch with me: josh@vitamin-j.com.

Copyright (c) 2008 Josh French, released under the MIT license
