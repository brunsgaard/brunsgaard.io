---
layout: post
title:  "Welcome to Jekyll!"
date:   2016-06-03 14:27:39 +0200
categories: jekyll update
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and
re-build the site to see your changes. You can rebuild the site in many
different ways, but the most common way is to run `jekyll serve`, which
launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows
the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front
matter. Take a look at the source for this post to get an idea about how it
works.

Jekyll also offers powerful support for code snippets:

{% highlight python %}
class RedisConnection:
    """Redis connection."""

    def __init__(self, reader, writer, *, encoding=None, loop=None):
        if loop is None:
            loop = asyncio.get_event_loop()
        self._reader = reader
        self._writer = writer
        self._loop = loop
        self._waiters = deque()
        self._parser = hiredis.Reader(protocolError=ProtocolError,
                                      replyError=ReplyError)
        self._reader_task = async_task(self._read_data(), loop=self._loop)
        self._db = 0
        self._closing = False
        self._closed = False
        self._close_waiter = create_future(loop=self._loop)
        self._reader_task.add_done_callback(self._close_waiter.set_result)
        self._in_transaction = None
        self._transaction_error = None  # XXX: never used?
        self._in_pubsub = 0
        self._pubsub_channels = coerced_keys_dict()
        self._pubsub_patterns = coerced_keys_dict()
        self._encoding = encoding

    def __repr__(self):
        return '<RedisConnection [db:{}]>'.format(self._db)

    @asyncio.coroutine
    def _read_data(self):
        """Response reader task."""
        while not self._reader.at_eof():
            try:
                data = yield from self._reader.read(MAX_CHUNK_SIZE)
            except asyncio.CancelledError:
                break
            except Exception as exc:
                # XXX: for QUIT command connection error can be received
                #       before response
                logger.error("Exception on data read %r", exc, exc_info=True)
                break
            self._parser.feed(data)
            while True:
                try:
                    obj = self._parser.gets()
                except ProtocolError as exc:
                    # ProtocolError is fatal
                    # so connection must be closed
                    self._closing = True
                    self._loop.call_soon(self._do_close, exc)
                    if self._in_transaction is not None:
                        self._transaction_error = exc
                    return
                else:
                    if obj is False:
                        break
                    if self._in_pubsub:
                        self._process_pubsub(obj)
                    else:
                        self._process_data(obj)
        self._closing = True
        self._loop.call_soon(self._do_close, None)
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the
most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub
repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll
Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
