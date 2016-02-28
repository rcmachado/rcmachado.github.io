---
layout: post
title: PHP & Memcached Gotchas
permalink: /:categories/:year/:month/:day/:title.html
categories:
- php
- memcache
---

Setting up a memcached session backend for PHP could be more tricky than it
sounds.

---

At a first glance is not that difficult to use memcached as session backend for
PHP. You only needs to choose one of the modules available (I prefer the
[memcached][memcached-ext], but you can use [memcache][memcache-ext]) if you want.

There is a lot of posts out there explaining how to configure [memcached as a session
backend][search_memcached_php_session], but just to recap:

1. Make sure you have memcached installed and running (a little obvious, but who knows?)
2. Change the following directives on your PHP ini:

{% highlight ini %}
; use 'memcache' if you choosed the memcache extension
session.save_handler = 'memcached'
session.save_path = '127.0.0.1:11211?persistent=1&timeout=2'
{% endhighlight %}

This should be enought. But what if we want to customize more parameters?

## Customizing session lifetime

Supose we want our session to last a little longer - let's say, for a week. The first
thing is to configure the session cookie to not expire when user closes the browser:

{% highlight ini %}
; set session cookie lifetime to 1 day
session.cookie_lifetime = 86400
{% endhighlight %}

But memcached will expire our sessions before that if we don't also change the
[session.gc_maxlifetime][] setting:

{% highlight ini %}
; consider session as garbage after 1 day
session.gc_maxlifetime = 86400
{% endhighlight %}

With these two settings, we extended our session duration to 1 day. Obviously, our session
keys can expire before one day. If this is happening, try to allocate more memory to your
memcached instances.

Everything is fine now, but what if we want longer sessions?

## Extending session lifetime - the problem

Lets assume that we want to keep the session approximately for 2 months. Doing the math,
this corresponds to ~5184000 seconds (60 * 60 * 24 * 30 * 2). We change our settings againg:

{% highlight ini %}
session.cookie_lifetime = 5184000
session.gc_maxlifetime = 5184000
{% endhighlight %}

The expected result is to keep the sessions for 2 months, but this is not what happened.
Mysteriously, our session data disapeared for memcached server. How is that possible?

## The gotcha

The gotcha - a.k.a. the feature that brings more problems than help - is that memcached
server is considering that expire time as an absolute timestamp instead of an offset from
current time. [Its by the design][memcached-protocol-79]:

    Some commands involve a client sending some kind of expiration time
    (relative to an item or to an operation requested by the client) to
    the server. In all such cases, the actual value sent may either be
    Unix time (number of seconds since January 1, 1970, as a 32-bit
    value), or a number of seconds starting from current time. In the
    latter case, this number of seconds may not exceed 60*60*24*30 (number
    of seconds in 30 days); if the number sent by a client is larger than
    that, the server will consider it to be real Unix time value rather
    than an offset from current time.

## Don't do that

Don't use session expiration times greater that 1 month (60*60*24*30, or 2592000 seconds).
Most of the time, this is not necessary nor recommended. If your users don't come back to
your site in 1 month it's better to force them to login again, as a security measure.

If you really think you need that your sessions last that much time, think again: probably
you're storing the wrong type of data on session. If you want to use memcached as a cache
backend don't do that using the session mechanism - it's very inefficient in that way. Store
and retrieve values explicitly.

## A not recommended workaround

Sometimes, specially in legacy projects, we simply can't change the way the system works for
whatever reason. If is that the case, there is a simple workaround that you could try: instead
of setting session configuration on ini file, you could change them using the [ini_set][]
function, forcing a timestamp instead of an offset from current time:

{% highlight php %}
<?php
$now = time();
ini_set('session.cookie_lifetime' $now + 5184000);
ini_set('session.gc_maxlifetime', $now + 5184000);
{% endhighlight %}

But as I said before: reconsider you use of sessions: maybe you are just trying to put the data
in the wrong place.

[memcached-ext]: http://php.net/memcached "Memcached PHP extension"
[memcache-ext]: http://php.net/memcache "Memcache PHP extension"
[search_memcached_php_session]: https://duckduckgo.com/?q=php+session+memcache+tutorial
[session.gc_maxlifetime]: http://php.net/manual/en/session.configuration.php#ini.session.gc-maxlifetime
[memcached-protocol-79]: https://github.com/memcached/memcached/blob/e31a591210311d0658a90a86f71563fa6d7b095c/doc/protocol.txt#L79
[ini_set]: http://php.net/ini_set
