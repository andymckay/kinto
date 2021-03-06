.. _tutorial:

First steps with Kinto
######################

There are actually two kinds of applications where *Kinto* is
particulary relevant as a storage backend:

  - Sync user data between devices;
  - Sync and share data between users, with fined-grained permissions.


Sync user data between devices
==============================

Let's say that we want to do a TodoMVC backend that will sync user
tasks between her devices.


In order to separate data between each user, we will use the default
*personal bucket*.

Unlike other buckets, the collections in the ``default`` bucket are created
implicitly.

Let us start with a really simple data model:

  - ``description``: A string describing the task
  - ``status``: The status of the task, (e.g. ``todo``, ``doing`` or ``done``).

And post a sample record in the ``tasks`` collection:

.. code-block:: http

    $ echo '{"data": {"description": "Write a tutorial explaining Kinto", "status": "todo"}}' | \
        http POST https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records \
             --auth 'user:password'

    POST /v1/buckets/default/collections/tasks/records HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 81
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "description": "Write a tutorial explaining Kinto",
            "status": "todo"
        }
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 253
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 15:31:55 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Write a tutorial explaining Kinto",
            "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
            "last_modified": 1434641515332,
            "status": "todo"
        },
        "permissions": {
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }


Let us fetch our new collection of tasks:

.. code-block:: http

    $ http GET https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records \
           --auth 'user:password'
    GET /v1/buckets/default/collections/tasks/records HTTP/1.1
    Accept: */*
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert, Next-Page, Total-Records, Last-Modified, ETag
    Backoff: 10
    Connection: keep-alive
    Content-Length: 152
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 15:34:04 GMT
    ETag: "1434641474977"
    Last-Modified: Thu, 18 Jun 2015 15:31:14 GMT
    Server: nginx/1.4.6 (Ubuntu)
    Total-Records: 1

    {
        "data": [
            {
                "description": "Write a tutorial explaining Kinto",
                "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
                "last_modified": 1434641515332,
                "status": "todo"
            }
        ]
    }


We can obviously also update one of our tasks, using its ``id``:

.. code-block:: http

    $ echo '{"data": {"status": "doing"}}' | \
         http PATCH https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 \
              -v  --auth 'user:password'

    PATCH /v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 30
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "status": "doing"
        }
    }

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 254
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 15:50:03 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Write a tutorial explaining Kinto",
            "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
            "last_modified": 1434642603605,
            "status": "doing"
        },
        "permissions": {
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }

There you should ask yourself, what happens if another device already
updated the record in between, will this request overwrite changes?

With the request shown above, yes it will.

In case you want the server to reject changes if the record was modified
in the interim, you must send the ``If-Match`` header.

In the ``If-Match`` header, you can send either the ``ETag`` header value you
obtained while fetching the collection, or the value of the ``last_modified``
data field you had for this record.

Let's try to modify the record using an obsolete value of ``ETag`` (obtained
while we fetched the collection):

.. code-block:: http

    $ echo '{"data": {"status": "done"}}' | \
        http PATCH https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 \
            If-Match:'"1434641515332"' \
            -v  --auth 'user:password'

    PATCH /v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 29
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    If-Match: "1434641515332"
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "status": "done"
        }
    }

    HTTP/1.1 412 Precondition Failed
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Connection: keep-alive
    Content-Length: 98
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:08:31 GMT
    ETag: "1434642603605"
    Last-Modified: Thu, 18 Jun 2015 15:50:03 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "code": 412,
        "errno": 114,
        "error": "Precondition Failed",
        "message": "Resource was modified meanwhile"
    }

As expected here, the server rejects the modification with a ``412 Precondition Failed``
error response.

In order to update this record safely, we can fetch the last version of this single record,
and merge attributes locally:

.. code-block:: http

    $ http GET https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 \
           -v  --auth 'user:password'

    GET /v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 HTTP/1.1
    Accept: */*
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2


    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert, Last-Modified, ETag
    Backoff: 10
    Connection: keep-alive
    Content-Length: 254
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:13:21 GMT
    ETag: "1434641474977"
    Last-Modified: Thu, 18 Jun 2015 15:31:14 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Write a tutorial explaining Kinto",
            "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
            "last_modified": 1434642603605,
            "status": "doing"
        },
        "permissions": {
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }

The strategy to merge changes locally are up to client, and might depend
on the client specifications. *Three-way merge* is possible when changes do
not affect the same fields, or if both objects are equal. Prompting the user
to decide what version should be kept might also be an option.

Once merged, we can send back again our modifications using the last
record ``ETag`` value:

.. code-block:: http

    $ echo '{"data": {"status": "done"}}' | \
        http PATCH https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 \
            If-Match:'"1434642603605"' \
            -v  --auth 'user:password'

    PATCH /v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 29
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    If-Match: "1434642603605"
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "status": "done"
        }
    }

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 253
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:21:16 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Write a tutorial explaining Kinto",
            "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
            "last_modified": 1434644476758,
            "status": "done"
        },
        "permissions": {
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }


You can also delete the record and use the same mechanism to avoid conflicts:

.. code-block:: http

    $ http DELETE https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 \
           If-Match:'"1434644476758"' \
           -v  --auth 'user:password'

    DELETE /v1/buckets/default/collections/tasks/records/23eed462-c063-4ae0-81b0-8bf2210bfe86 HTTP/1.1
    Accept: */*
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 0
    Host: kinto.dev.mozaws.net
    If-Match: "1434644476758"
    User-Agent: HTTPie/0.9.2

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 99
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:27:03 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "deleted": true,
            "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
            "last_modified": 1434644823180
        }
    }


Likewise, we can query the list of changes (updates and deletions) that occured
since we had fetched the collection.

Just add the ``_since`` querystring filter, using the value of any ``ETag`` (or ``last_modified`` data field):

.. code-block:: http

    $ http GET https://kinto.dev.mozaws.net/v1/buckets/default/collections/tasks/records?_since=1434642603605 \
           -v  --auth 'user:password'

    GET /v1/buckets/default/collections/tasks/records?_since=1434642603605 HTTP/1.1
    Accept: */*
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2


    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert, Next-Page, Total-Records, Last-Modified, ETag
    Backoff: 10
    Connection: keep-alive
    Content-Length: 101
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:29:54 GMT
    ETag: "1434641474977"
    Last-Modified: Thu, 18 Jun 2015 15:31:14 GMT
    Server: nginx/1.4.6 (Ubuntu)
    Total-Records: 0

    {
        "data": [
            {
                "deleted": true,
                "id": "23eed462-c063-4ae0-81b0-8bf2210bfe86",
                "last_modified": 1434644823180
            }
        ]
    }


The list will be empty if no change occured. Instead, you can also use the
``If-None-Match`` header with the last ``ETag`` value in order to obtain a
``304 Not Modified`` response if nothing changed.


Sync and share data between users
=================================

Instead of using the *personal bucket*, we will create a specific bucket ``todo``
for the application.

.. code-block:: http

    $ echo '{"data": {}}' | http PUT https://kinto.dev.mozaws.net/v1/buckets/todo \
        -v --auth 'user:password'

    PUT /v1/buckets/todo HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 13
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {}
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 155
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:33:17 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "todo",
            "last_modified": 1434645197868
        },
        "permissions": {
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }

By default, the creator is the only administrator (see ``write`` permission).

You will now have to define permissions to introduce collaboration.

In our case, we want people to be able create and share tasks, so
we will create a collection ``tasks`` with the ``record:create`` permission for
authenticated users (i.e. ``system.Authenticated``):

.. code-block:: http

    $ echo '{"data": {}, "permissions": {"record:create": ["system.Authenticated"]}}' | \
        http PUT https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks \
            -v --auth 'user:password'

    PUT /v1/buckets/todo/collections/tasks HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 73
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {},
        "permissions": {
            "record:create": [
                "system.Authenticated"
            ]
        }
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 197
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:37:48 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "tasks",
            "last_modified": 1434645468367
        },
        "permissions": {
            "record:create": [
                "system.Authenticated"
            ],
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }

.. note::

   As you may noticed, you are automatically added to the ``write``
   permission of any objects you are creating.


Now Alice can create a task in this collection:

.. code-block:: http

    $ echo '{"data": {"description": "Alice task", "status": "todo"}}' | \
        http POST https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks/records \
        -v --auth 'alice:alicepassword'

    POST /v1/buckets/todo/collections/tasks/records HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic YWxpY2U6YWxpY2VwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 59
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "description": "Alice task",
            "status": "todo"
        }
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 231
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:41:50 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Alice task",
            "id": "2fa91620-f4fa-412e-aee0-957a7ad2dc0e",
            "last_modified": 1434645840590,
            "status": "todo"
        },
        "permissions": {
            "write": [
                "basicauth:9be2b51de8544fbed4539382d0885f8643c0185c90fb23201d7bbe86d70b4a44"
            ]
        }
    }

And Bob can also create a task:

.. code-block:: http

    $ echo '{"data": {"description": "Bob new task", "status": "todo"}}' | \
        http POST https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks/records \
        -v --auth 'bob:bobpassword'

    POST /v1/buckets/todo/collections/tasks/records HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic Ym9iOmJvYnBhc3N3b3Jk
    Connection: keep-alive
    Content-Length: 60
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "description": "Bob new task",
            "status": "todo"
        }
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 232
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:44:39 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "description": "Bob new task",
            "id": "10afe152-b5bb-4aff-b77e-10be44587057",
            "last_modified": 1434645879088,
            "status": "todo"
        },
        "permissions": {
            "write": [
                "basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148"
            ]
        }
    }


If Alice wants to share a task with Bob, she can give him the ``read`` permission
on her records:

.. code-block:: http

    $ echo '{
        "permissions": {
            "read": ["basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148"]
        }
    }' | \
    http PATCH https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks/records/2fa91620-f4fa-412e-aee0-957a7ad2dc0e \
        -v --auth 'alice:alicepassword'

    PATCH /v1/buckets/todo/collections/tasks/records/2fa91620-f4fa-412e-aee0-957a7ad2dc0e HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic YWxpY2U6YWxpY2VwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 118
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "permissions": {
            "read": [
                "basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148"
            ]
        }
    }

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 273
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:50:57 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "2fa91620-f4fa-412e-aee0-957a7ad2dc0e",
            "last_modified": 1434646257547
        },
        "permissions": {
            "read": [
                "basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148"
            ],
            "write": [
                "basicauth:9be2b51de8544fbed4539382d0885f8643c0185c90fb23201d7bbe86d70b4a44"
            ]
        }
    }


Here we share individual records, and nobody (except its creator) can obtain the
collection records. For example, a ``read`` permission could be added on
the collection to allow authenticated users to list the whole list.

.. notes::

    Currently, *Kinto* does not support the use-case where the whole collection is
    private and each user obtains a list containing only the records where she
    has read access.

    **This is on top of our priorities!**


Working with groups
===================

To go further, you may want to allow users to share data with a group
of users.

Let's add the permission for authenticated users to create groups in the ``todo``
bucket:

.. code-block:: http

    $ echo '{"permissions": {"group:create": ["system.Authenticated"]}}' | \
        http PATCH https://kinto.dev.mozaws.net/v1/buckets/todo \
            -v --auth 'user:password'

    PATCH /v1/buckets/todo HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic dXNlcjpwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 72
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "permissions": {
            "group:create": [
                "system.Authenticated"
            ]
        }
    }

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 195
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 16:59:29 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "todo",
            "last_modified": 1434646769990
        },
        "permissions": {
            "group:create": [
                "system.Authenticated"
            ],
            "write": [
                "basicauth:10ea4e5fbf849196a4fe8a9c250b737dd5ef17abbeb8f99692d62828465a9823"
            ]
        }
    }

Now Alice can create a group of her friends Bob and Mary:

.. code-block:: http

    $ echo '{"data": {
        "members": ["basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148",
                    "basicauth:8d1661a89bd2670f3c42616e3527fa30521743e4b9825fa4ea05adc45ef695b6"]
    }}' | http PUT https://kinto.dev.mozaws.net/v1/buckets/todo/groups/alice-friends \
        -v --auth 'alice:alicepassword'

    PUT /v1/buckets/todo/groups/alice-friends HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic YWxpY2U6YWxpY2VwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 180
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "data": {
            "members": [
                "basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148",
                "basicauth:8d1661a89bd2670f3c42616e3527fa30521743e4b9825fa4ea05adc45ef695b6"
            ]
        }
    }

    HTTP/1.1 201 Created
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 330
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 17:03:24 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "alice-friends",
            "last_modified": 1434647004644,
            "members": [
                "basicauth:a103c2e714a04615783de8a03fef1c7fee221214387dd07993bb9aed1f2f2148",
                "basicauth:8d1661a89bd2670f3c42616e3527fa30521743e4b9825fa4ea05adc45ef695b6"
            ]
        },
        "permissions": {
            "write": [
                "basicauth:9be2b51de8544fbed4539382d0885f8643c0185c90fb23201d7bbe86d70b4a44"
            ]
        }
    }

Consequently, Alice can share her records directly with her group of friends:

.. code-block:: http

    $ echo '{
        "permissions": {
            "read": ["/buckets/todo/groups/alice-friends"]
        }
    }' | \
    http PATCH https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks/records/2fa91620-f4fa-412e-aee0-957a7ad2dc0e \
        -v --auth 'alice:alicepassword'

    PATCH /v1/buckets/todo/collections/tasks/records/2fa91620-f4fa-412e-aee0-957a7ad2dc0e HTTP/1.1
    Accept: application/json
    Accept-Encoding: gzip, deflate
    Authorization: Basic YWxpY2U6YWxpY2VwYXNzd29yZA==
    Connection: keep-alive
    Content-Length: 122
    Content-Type: application/json
    Host: kinto.dev.mozaws.net
    User-Agent: HTTPie/0.9.2

    {
        "permissions": {
            "read": [
                "/buckets/todo/groups/alice-friends"
            ]
        }
    }

    HTTP/1.1 200 OK
    Access-Control-Expose-Headers: Backoff, Retry-After, Alert
    Backoff: 10
    Connection: keep-alive
    Content-Length: 237
    Content-Type: application/json; charset=UTF-8
    Date: Thu, 18 Jun 2015 17:06:09 GMT
    Server: nginx/1.4.6 (Ubuntu)

    {
        "data": {
            "id": "2fa91620-f4fa-412e-aee0-957a7ad2dc0e",
            "last_modified": 1434647169157
        },
        "permissions": {
            "read": [
                "/buckets/todo/groups/alice-friends"
            ],
            "write": [
                "basicauth:9be2b51de8544fbed4539382d0885f8643c0185c90fb23201d7bbe86d70b4a44"
            ]
        }
    }

And now, Mary can access the record:

.. code-block:: http

    $ http GET https://kinto.dev.mozaws.net/v1/buckets/todo/collections/tasks/records/2fa91620-f4fa-412e-aee0-957a7ad2dc0e \
        -v --auth 'mary:marypassword'


.. note::

    The records of the personal bucket can also be shared! In order to obtain
    its id, just use ``GET /buckets/default`` and then share its content using
    the full URL (e.g. ``/buckets/b86b26b8-be36-4eaa-9ed9-2e6de63a5252``)!


Conclusion
==========

In this tutorial, you have seen some of the concepts exposed by *Kinto*:

- Using the default personal user bucket;
- Handling synchronization and conflicts;
- Creating a bucket to share data between users;
- Creating groups, collections and records;
- Modifying objects permissions, for users and groups;

More details about permissions, HTTP API headers and status codes.

.. notes::

    We plan to improve our documentation and make sure it is as easy as
    possible to get started with *Kinto*.

    Please do not hesitate to :ref:`give us feedback <contributing>`, and if you are
    interested to improve it, join us!
