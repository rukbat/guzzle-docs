=====================
Using Request objects
=====================

HTTP request messages
---------------------

Request objects are all about building an HTTP message. Each part of an HTTP request message can be set individually
using methods on the request object or set in bulk using the ``setUrl()`` method. Here's the format of an HTTP request
with each part of the request referencing the method used to change it::

    PUT(a) /path(b)?query=123(c) HTTP/1.1(d)
    X-Header(e): header
    Content-Length(e): 4

    data(f)

+-------------------------+---------------------------------------------------------------------------------+
| a. **Method**           | The request method can only be set when instantiating a request                 |
+-------------------------+---------------------------------------------------------------------------------+
| b. **Path**             | ``$request->setPath('/path');``                                                 |
+-------------------------+---------------------------------------------------------------------------------+
| c. **Query**            | ``$request->getQuery()->set('query', '123');``                                  |
+-------------------------+---------------------------------------------------------------------------------+
| d. **Protocol version** | ``$request->setProtocolVersion('1.1');``                                        |
+-------------------------+---------------------------------------------------------------------------------+
| e. **Header**           | ``$request->setHeader('X-Header', 'header');``                                  |
+-------------------------+---------------------------------------------------------------------------------+
| f. **Entity Body**      |  ``$request->setBody('data'); // Only available with PUT, POST, PATCH, DELETE`` |
+-------------------------+---------------------------------------------------------------------------------+

Creating requests with a client
-------------------------------

Client objects are responsible for creating HTTP request objects.

.. note::

    The job of actually creating the request objects is delegated from a client to a
    ``Guzzle\Http\Message\RequestFactoryInterface`` object owned by a client. You can modify the request classes
    instantiated by a client by injecting a custom request factory into the client (using ``setRequestFactory()``).

GET requests
~~~~~~~~~~~~

`GET requests <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.3>`_ are the most common form of HTTP
requests. When you visit a website in your browser, the HTML of the website is downloaded using a GET request. GET
requests are idempotent requests that are typically used to download content (an entity) identified by a request URL.

.. code-block:: php

    use Guzzle\Http\Client;

    $client = new Client();

    // Create a request that has a query string and an X-Foo header
    $request = $client->get('http://www.amazon.com?a=1', array('X-Foo' => 'Bar'));

    // Send the request and get the response
    $response = $request->send();

The above code sample will send the following GET request::

    GET /?a=1 HTTP/1.1
    Host: www.amazon.com
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15
    X-Foo: Bar

HEAD requests
~~~~~~~~~~~~~

`HEAD requests <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.4>`_ work exactly like GET requests except
that they do not actually download the response body (entity) of the response message. HEAD requests are useful for
retrieving meta information about an entity identified by a Request-URI.

.. code-block:: php

    $client = new Guzzle\Http\Client();
    $request = $client->head('http://www.amazon.com');
    $response = $request->send();
    echo $response->getContentLength();
    // >>> Will output the Content-Length header value

The above code sample will send the following HEAD request::

    HEAD / HTTP/1.1
    Host: www.amazon.com
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15

DELETE requests
~~~~~~~~~~~~~~~

A `DELETE method <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.7>`_ requests that the origin server
delete the resource identified by the Request-URI.

.. code-block:: php

    $client = new Guzzle\Http\Client();
    $request = $client->delete('http://example.com');
    $response = $request->send();

The above code sample will send the following DELETE request::

    DELETE / HTTP/1.1
    Host: example.com
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15


POST requests
~~~~~~~~~~~~~

While `POST requests <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.5>`_ can be used for a number of
reasons, POST requests are often used when submitting HTML form data to a website. POST request can include an entity
body in the HTTP request.

POST requests in Guzzle are sent with an ``application/x-www-form-urlencoded`` Content-Type header if POST fields are
present but no files are being sent in the POST. If files are specified in the POST request, then the Content-Type
header will become ``multipart/form-data``.

The ``post()`` method of a client object accepts three arguments: the URL, optional headers, and the post fields. To
send files in the POST request, prepend the ``@`` symbol to the array value (just like you would if you were using the
PHP ``curl_setopt`` function).

Here's how to create a multipart/form-data POST request containing files and fields:

.. code-block:: php

    $request = $client->post('http://httpbin.org/post', null, array(
        'custom_field' => 'my custom value',
        'file_field'   => '@/path/to/file.xml'
    ));

    $response = $request->send();

.. note::

    Remember to **always** sanitize user input when sending POST requests:

    .. code-block:: php

        // Prevent users from accessing sensitive files by sanitizing input
        $_POST = array('firstname' => '@/etc/passwd');
        $request = $client->post('http://www.example.com', null, array (
            'firstname' => str_replace('@', '', $_POST['firstname'])
        ));

You can alternatively build up the contents of a POST request.

.. code-block:: php

    $request = $client->post('http://httpbin.org/post')
        ->addPostField('custom_field', 'my custom value')
        ->addPostFile('file', '/path/to/file.xml');

    $response = $request->send();

Raw POST data
^^^^^^^^^^^^^

POST requests can also contain raw POST data that is not related to HTML forms.

.. code-block:: php

    $request = $client->post('http://httpbin.org/post', null, 'this is the body');
    $response = $request->send();

The above code sample will send the following POST request::

    POST /post HTTP/1.1
    Host: httpbin.org
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15
    Content-Length: 16

    this is the body

You can set the body of POST request using the ``setBody()`` method of the
``Guzzle\Http\Message\EntityEnclosingRequest`` object. This method accepts a string, a resource returned from
``fopen``, or a ``Guzzle\Http\EntityBodyInterface`` object.

.. code-block:: php

    $request = $client->post('http://httpbin.org/post');
    // Set the body of the POST to stream the contents of /path/to/larg_body.txt
    $request->setBody(fopen('/path/to/large_body.txt', 'r'));
    $response = $request->send();

PUT requests
~~~~~~~~~~~~

The `PUT method <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.6>`_ requests that the enclosed entity be
stored under the supplied Request-URI. PUT requests are similar to POST requests in that they both can send an entity
body in the request message.

The body of a PUT request (any any ``Guzzle\Http\Message\EntityEnclosingRequestInterface`` object) is always stored as
a ``Guzzle\Http\Message\EntityBodyInterface`` object. This allows a great deal of flexibility when sending data to a
remote server. For example, you can stream the contents of a stream returned by fopen, stream the contents of a
callback function, or simply send a string of data.

.. code-block:: php

    $request = $client->put('http://httpbin.org/put', null, 'this is the body');
    $response = $request->send();

The above code sample will send the following PUT request::

    PUT /put HTTP/1.1
    Host: httpbin.org
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15
    Content-Length: 16

    this is the body

Just like with POST, PATH, and DELETE requests, you can set the body of a PUT request using the ``setBody()`` method.

.. code-block:: php

    $request = $client->put('http://httpbin.org/put');
    $request->setBody(fopen('/path/to/large_body.txt', 'r'));
    $response = $request->send();

PATCH requests
~~~~~~~~~~~~~~

`PATCH requests <http://tools.ietf.org/html/rfc5789>`_ are used to modify a resource.

.. code-block:: php

    $request = $client->put('http://httpbin.org', null, 'this is the body');
    $response = $request->send();

The above code sample will send the following PATCH request::

    PATCH / HTTP/1.1
    Host: httpbin.org
    User-Agent: Guzzle/3.3.1 curl/7.21.4 PHP/5.3.15
    Content-Length: 16

    this is the body

OPTIONS requests
~~~~~~~~~~~~~~~~

The `OPTIONS method <http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.2>`_ represents a request for
information about the communication options available on the request/response chain identified by the Request-URI.

.. code-block:: php

    $request = $client->options('http://httpbin.org');
    $response = $request->send();

    // Check if the PUT method is supported by this resource
    var_export($response->isMethodAllows('PUT'));

Custom requests
~~~~~~~~~~~~~~~

You can create custom HTTP requests that use non-standard HTTP methods using the ``createRequest()`` method of a
client object.

.. code-block:: php

    $request = $client->createRequest('COPY', 'http://example.com/foo', array(
        'Destination' => 'http://example.com/bar',
        'Overwrite'   => 'T'
    ));
    $response = $request->send();

Query string parameters
-----------------------

Query string parameters of a request are owned by a request's ``Guzzle\Http\Query`` object that is accessible by
calling ``$request->getQuery()``. The Query class extends from ``Guzzle\Common\Collection`` and allows you to set one
or more query string parameters as key value pairs. You can set a parameter on a Query object using the
``set($key, $value)`` method. Any previously specified value for a key will be overwritten when using ``set()``. Use
``add($key, $value)`` to add a value to query string object, and in the event of a collision with an existing value at
a specific key, the value will be converted to an array that contains all of the previously set values.

.. code-block:: php

    $request = new Guzzle\Http\Message\Request('GET', 'http://www.example.com?foo=bar&abc=123');

    $query = $request->getQuery();
    echo "{$query}\n";
    // >>> foo=bar&abc=123

    $query->remove('abc');
    echo "{$query}\n";
    // >>> foo=bar

    $query->set('foo', 'baz');
    echo "{$query}\n";
    // >>> foo=baz

    $query->add('foo', 'bar');
    echo "{$query}\n";
    // >>> foo%5B0%5D=baz&foo%5B1%5D=bar

Whoah! What happened there? When ``foo=bar`` was added to the existing ``foo=baz`` query string parameter, the
aggregator associated with the Query object was used to help convert multi-value query string parameters into a string.
Let's disable URL-encoding to better see what's happening.

.. code-block:: php

    $query->useUrlEncoding(false);
    echo "{$query}\n";
    // >>> foo[0]=baz&foo[1]=bar

.. note::

    URL encoding can be disabled by passing false, enabled by passing true, set to use RFC 1738 by passing
    ``Query::FORM_URLENCODED`` (internally uses PHP's ``urlencode`` function), or set to RFC 3986 by passing
    ``Query::RFC_3986`` (this is the default and internally uses PHP's ``rawurlencode`` function).

As you can see, the multiple values were converted into query string parameters following the default PHP convention of
adding numerically indexed square bracket suffixes to each key (``foo[0]=baz&foo[1]=bar``). The strategy used to convert
mutli-value parameters into a string can be customized using the ``setAggregator()`` method of the Query class. Guzzle
ships with the following query string aggregators by default:

1. ``Guzzle\Http\QueryAggregator\PhpAggregator``: Aggregates using PHP style brackets (e.g. ``foo[0]=baz&foo[1]=bar``)
2. ``Guzzle\Http\QueryAggregator\DuplicateAggregator``: Performs no aggregation and allows for key value pairs to be
   repeated in a URL (e.g. ``foo=baz&foo=bar``)
3. ``Guzzle\Http\QueryAggregator\CommaAggregator``: Aggregates using commas (e.g. ``foo=baz,bar``)

.. _http-message-headers:

HTTP Message Headers
--------------------

HTTP message headers are case insensitive, multiple occurrences of any header can be present in an HTTP message
(whether it's valid or not), and some servers require specific casing of particular headers. Because of this, request
and response headers are stored in ``Guzzle\Http\Message\Header`` objects. The Header object can be cast as a string,
counted, or iterated to retrieve each value from the header. Casting a Header object to a string will return all of
the header values concatenated together using a glue string (typically ", ").

A request (and response) object have several methods that allow you to retrieve and modify headers.

* ``getHeaders()``: Get all of the headers of a message as a ``Guzzle\Common\Collection`` object.
* ``getHeader($header)``: Get a specific header from a message. If the header exists, you'll get a
  ``Guzzle\Http\Message\Header`` object. If the header does not exist, this methods returns ``null``.
* ``hasHeader($header)``: Returns true or false based on if the message has a particular header.
* ``setHeader($header, $value)``: Set a header value and overwrite any previously set value for this header.
* ``addHeader($header, $value)``: Add a header with a particular name. If a previous value was already set by the same,
  then the header will contain multiple values.
* ``removeHeader($header)``: Remove a header by name from the message.

.. code-block:: php

    $request = new Request('GET', 'http://httpbin.com/cookies');
    // addHeader will set and append to any existing header values
    $request->addHeader('Foo', 'bar');
    $request->addHeader('foo', 'baz');
    // setHeader overwrites any existing values
    $request->setHeader('Test', '123');

    // Request headers can be cast as a string
    echo $request->getHeader('Foo');
    // >>> bar, baz
    echo $request->getHeader('Test');
    // >>> 123

    // You can count the number of headers of a particular case insensitive name
    echo count($request->getHeader('foO'));
    // >>> 2

    // You can iterate over Header objects
    foreach ($request->getHeader('foo') as $header) {
        echo $header . "\n";
    }

    // You can get all of the request headers as a Guzzle\Common\Collection object
    $headers = $request->getHeaders();

    // Missing headers return NULL
    var_export($request->getHeader('Missing'));
    // >>> null

    // You can see all of the different variations of a header by calling raw() on the Header
    var_export($request->getHeader('foo')->raw());

Setting the body of a request
-----------------------------

Requests that can send a body (e.g. PUT, POST, DELETE, PATCH) are instances of
``Guzzle\Http\Message\EntityEnclosingRequestInterface``. Entity enclosing requests contain several methods that allow
you to specify the body to send with a request.

Use the ``setBody()`` method of a request to set the body that will be sent with a request. This method accepts a
string, a resource returned by ``fopen()``, an array, or an instance of ``Guzzle\Http\EntityBodyInterface``. The body
will then be streamed from the underlying ``EntityBodyInterface`` object owned by the request. When setting the body
of the request, you can optionally specify a Content-Type header and whether or not to force the request to use
chunked Transfer-Encoding.

.. code-block:: php

    $request = $client->put('/user.json');
    $request->setBody('{"foo":"baz"}', 'application/json');

In the above example, the Content-Length of the body can be easily determined and will be automatically added to the
request. If the Content-Length cannot be determined (i.e. using a PHP ``http://`` stream), then the request will gain
the ``Transfer-Encoding: chunked`` header.

.. code-block:: php

    $request = $client->put('/user.json');
    $request->setBody(fopen('http://httpbin.org/get', 'r'));

    // The Content-Type was guessed based on the path of the request
    echo $request->getHeader('Content-Type');
    // >>> application/json

    // The Content-Length could not be determined
    echo $request->getHeader('Transfer-Encoding');
    // >>> chunked

See :doc:`/http-client/entity-bodies` for more information on entity bodies.

Expect: 100-Continue header
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``Expect: 100-Continue`` header is used to help a client prevent sending a large payload to a server that will
reject the request. This allows clients to fail fast rather than waste bandwidth sending an erroneous payload. Guzzle
will automatically add the ``Expect: 100-Continue`` header to a request when the size of the payload exceeds 1MB or if
the body of the request is not seekable (this helps to prevent errors when a non-seekable body request is redirected).

POST fields and files
~~~~~~~~~~~~~~~~~~~~~

Any entity enclosing request can send POST style fields and files. This includes POST, PUT, PATCH, and DELETE requests.
Any request that has set POST fields or files will use cURL's POST message functionality.

.. code-block:: php

    $request = $client->post('/post');
    // Set an overwrite any previously specified value
    $request->setPostField('foo', 'bar');
    // Append a value to any existing values
    $request->addPostField('foo', 'baz');
    // Remove a POST field by name
    $request->removePostField('fizz');

    // Add a file to upload (forces multipart/form-data)
    $request->addPostFile('my_file', '/path/to/file', 'plain/text');
    // Remove a POST file by POST key name
    $request->removePostFile('my_other_file');

.. tip::

    Adding a large number of POST fields to a POST request is faster if you use the ``addPostFields()`` method so that
    you can add and process multiple fields with a single call. Adding multiple POST files is also faster using
    ``addPostFiles()``.

Working with cookies
--------------------

Cookies can be modified and retrieved from a request using the following methods:

.. code-block:: php

    $request->addCookie($name, $value);
    $request->removeCookie($name);
    $value = $request->getCookie($name);
    $valueArray = $request->getCookies();

Use the :doc:`cookie plugin </plugins/cookie-plugin>` if you need to reuse cookies between requests.

Changing where a response is downloaded
----------------------------------------

When a request is sent, the body of the response will be stored in a PHP temp stream by default. You can change the
location in which the response will be downloaded using ``$request->setResponseBody($body)``. This can be useful for
downloading the contents of a URL to a specific file.

.. code-block:: php

    $body = fopen('/tmp/large_file.mov', 'w');
    $request = $this->client->get('http://example.com/large.mov');
    $request->setResponseBody($body);
    $request->send();

    var_export(file_exists('/tmp/large_file.mov'));
    // >>> true

You can more easily specify the name of a file to save the contents of the response to by passing a string to
``setResponseBody()``.

.. code-block:: php

    $request = $this->client->get('http://example.com/large.mov');
    $request->setResponseBody('/tmp/large_file.mov');
    $request->send();

    var_export(file_exists('/tmp/large_file.mov'));
    // >>> true

Custom cURL options
-------------------

Most of the functionality implemented in the libcurl bindings has been simplified and abstracted by Guzzle. Developers
who need access to `cURL specific functionality <http://www.php.net/curl_setopt>`_ that is not abstracted by Guzzle
(e.g. proxies and some SSL options) can still add cURL handle specific behavior to Guzzle HTTP requests by modifying
the cURL options collection of a request:

.. code-block:: php

    $request->getCurlOptions()->set(CURLOPT_SSL_VERIFYHOST, true);

Other special options that can be set in the ``curl.options`` array include:

+-------------------------+---------------------------------------------------------------------------------+
| debug                   | Adds verbose cURL output to a temp stream owned by the cURL handle object       |
+-------------------------+---------------------------------------------------------------------------------+
| progress                | Instructs cURL to emit events when IO events occur. This allows you to be       |
|                         | notified when bytes are transferred over the wire by subscribing to a request's |
|                         | ``curl.callback.read``, ``curl.callback.write``, and ``curl.callback.progress`` |
|                         | events.                                                                         |
+-------------------------+---------------------------------------------------------------------------------+

Timeouts
~~~~~~~~

cURL provides `several timeout options <http://www.php.net/curl_setopt>`_ that can be used to control the amount of
time a request will wait before timing out.

* ``CURLOPT_TIMEOUT``: The maximum number of seconds to allow cURL functions to execute.
* ``CURLOPT_TIMEOUT_MS``: The maximum number of milliseconds to allow cURL functions to execute.
* ``CURLOPT_CONNECTTIMEOUT``: The number of seconds to wait while trying to connect.
* ``CURLOPT_CONNECTTIMEOUT_MS``: The number of milliseconds to wait while trying to connect.

You can tell requests to stop waiting for a response after a given number of seconds with the CURLOPT_TIMEOUT
parameter:

.. code-block:: php

    $request = $client->get('http://www.example.com');
    // Time out after 5 seconds
    $request->getCurlOptions()->set(CURLOPT_TIMEOUT, 5);

Proxy settings
~~~~~~~~~~~~~~

Some corporate networks require that outbound HTTP requests are sent through a proxy. cURL offers several proxy
specific settings, but the most commonly using setting is ``CURLOPT_PROXY``.

.. code-block:: php

    $request = $client->get('http://www.example.com');
    $request->getCurlOptions()->set(CURLOPT_PROXY, 'tcp://127.0.0.1:8888');

Dealing with errors
-------------------

Exceptions
~~~~~~~~~~

Requests that receive a 4xx or 5xx response will throw a ``Guzzle\Http\Exception\BadResponseException``. More
specifically, 4xx errors throw a ``Guzzle\Http\Exception\ClientErrorResponseException``, and 5xx errors throw a
``Guzzle\Http\Exception\ServerErrorResponseException``. You can catch the specific exceptions or just catch the
BadResponseException to deal with either type of error. Here's an example of catching a generic BadResponseException:

.. code-block:: php

    try {
        $response = $client->get('/not_found.xml')->send();
    } catch (Guzzle\Http\Exception\BadResponseException $e) {
        echo 'Uh oh! ' . $e->getMessage();
    }

Throwing an exception when a 4xx or 5xx response is encountered is the default behavior of Guzzle requests. This
behavior can be overridden by adding an event listener with a higher priority than -255 that stops event propagation.
You can subscribe to ``request.error`` to receive notifications any time an unsuccessful response is received.

You can change the response that will be associated with the request by calling ``setResponse()`` on the
``$event['request']`` object passed into your listener, or by changing the ``$event['response']`` value of the
``Guzzle\Common\Event`` object that is passed to your listener. Transparently changing the response associated with a
request by modifying the event allows you to retry failed requests without complicating the code that uses the client.
This might be useful for sending requests to a web service that has expiring auth tokens. When a response shows that
your token has expired, you can get a new token, retry the request with the new token, and return the successful
response to the user.

Here's an example of retrying a request using updated authorization credentials when a 401 response is received,
overriding the response of the original request with the new response, and still allowing the default exception
behavior to be called when other non-200 response status codes are encountered:

.. code-block:: php

    // Add custom error handling to any request created by this client
    $client->getEventDispatcher()->addListener('request.error', function(Event $event) {

        if ($event['response']->getStatusCode() == 401) {

            $newRequest = $event['request']->clone();
            $newRequest->setHeader('X-Auth-Header', MyApplication::getNewAuthToken());
            $newResponse = $newRequest->send();

            // Set the response object of the request without firing more events
            $event['response'] = $newResponse;

            // You can also change the response and fire the normal chain of
            // events by calling $event['request']->setResponse($newResponse);

            // Stop other events from firing when you override 401 responses
            $event->stopPropagation();
        }

    });

cURL errors
~~~~~~~~~~~

Connection problems and cURL specific errors can also occur when transferring requests using Guzzle. When Guzzle
encounters cURL specific errors while transferring a single request, a ``Guzzle\Http\Exception\CurlException`` is
thrown with an informative error message and access to the cURL error message.

A ``Guzzle\Common\Exception\MultiTransferException`` exception is thrown when a cURL specific error occurs while
transferring multiple requests in parallel. You can then iterate over all of the exceptions encountered during the
transfer.