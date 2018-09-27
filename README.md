# Riprap

## Overview

A fixity-auditing microservice that addresses https://github.com/Islandora-CLAW/CLAW/issues/847. Developed as a successfor to Islandora 7.x's [Checksum Checker](https://github.com/Islandora/islandora_checksum_checker) module, it is intended primarily to be used with repositories compliant with the [Fedora API Specification](https://fedora.info/spec/), but can be used to provide fixity validation for other repositories as well (e.g., an [OCFL](https://ocfl.io/) repository). It periodcally requests fixity digests for resources from a repository and compares the digest with a previously request digest. It then persists the outocome of that comparison so the process can be repeated again. Riprap also provides a REST interface so that external applications (in Islandora's case, Drupal) can retrieve fixity checking event data for use in reports, etc.

![Overview](docs/images/overview.png)

Riprap generates and records fixity check events as described in the "Fixity, integrity, authenticity" section of the [PREMIS Data Dictionary for Preservation Metadata, Version 3.0](https://www.loc.gov/standards/premis/v3/premis-3-0-final.pdf). It can also record fixity information available during "ingestion" events and at the time of "deletion" events. The three typs of events are members of the Library of Congress' "[Event Type Collection](http://id.loc.gov/vocabulary/preservation/eventType/collection_PREMIS)" vocabulary:

* [ingestion](http://id.loc.gov/vocabulary/preservation/eventType/ing)
* [fixity check](http://id.loc.gov/vocabulary/preservation/eventType/fix)
* [deletion](http://id.loc.gov/vocabulary/preservation/eventType/del)

"fixity check" events are generated by Riprap, typically in a job scheduled via `cron`, but "ingestion" and "deletion" events are generated by external systems, which may persist information about those events at any time in Riprap's database via either its REST API or via an ActiveMQ message.

All events must have a value of `suc` or `fail` (using values from the PREMIS Event Outcome vocabulary (not yet published but will be soon).

## Current status

Riprap is still in early development, but all the major functional components are working using test/sampl data. Riprap is not yet ready for production but will be by December 2018.

## Requirements

* PHP 7.1.3 or higher
* [composer](https://getcomposer.org/)
* SQLite (other RDBMSs will be supported soon).

## Installation

1. Clone this git repository
1. `cd riprap`
1. `php composer.phar install` (or equivalent on your system, e.g., `./composer install`)

We will eventually support deployment via Ansible.

## Trying it out

If you want to play with Riprap, and you're on a Linux or OSX machine, you should not need to configure anything. Assuming you have sqlite installed, you should be able to run the `check_fixity` command against the sample data and local web server, and perform basic API requests requests as documented below. A couple of things you will want to know:

* the database created by Symfony will be in located at `[riprap directory]/var/data.db`
* Riprap will write its log to `/tmp/riprap.log`
* the test webserver runs on port 8000

### Generating sample events

As stated above, for now we use SQLite as our database. If you would like to generate some sample events, follow these instructions from within the `riprap` directory:

In `.env`, open an editor and add the following line in the `doctrine-bundle` section: `DATABASE_URL=sqlite:///%kernel.project_dir%/var/data.db`. Then run the following commands:

1. `rm var/data.db` (might not exist)
1. `rm src/Migrations/*` (might be empty)
1. `php bin/console -n make:migration`
1. `php bin/console -n doctrine:migrations:migrate`
1. `php bin/console -n doctrine:fixtures:load`

At this point you will have 20 rows in your database's `event` table. If you query the table you will see the following output:

`sqlite3 var/data.db`

```
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .headers on
sqlite> select * from event;
id|event_uuid|event_type|resource_id|datestamp|hash_algorithm|hash_value|event_outcome
1|2a40d01e-d0fc-49c0-8755-990c90e21f13|ing|http://localhost:8000/examplerepository/rest/1|2018-09-19 05:23:20|SHA-1|5a5b0f9b7d3f8fc84c3cef8fd8efaaa6c70d75ab|suc
2|27099e67-e355-4308-b618-e880900ee16a|ing|http://localhost:8000/examplerepository/rest/2|2018-09-19 05:23:20|SHA-1|b1d5781111d84f7b3fe45a0852e59758cd7a87e5|suc
3|b64d7dac-db2d-4984-b72e-46f6f33d1d0a|ing|http://localhost:8000/examplerepository/rest/3|2018-09-19 05:23:20|SHA-1|310b86e0b62b828562fc91c7be5380a992b2786a|suc
4|f1ff2644-6f6d-4765-84ee-ae2e6ea85b1b|ing|http://localhost:8000/examplerepository/rest/4|2018-09-19 05:23:20|SHA-1|08a35293e09f508494096c1c1b3819edb9df50db|suc
5|59d47475-3c47-412e-a94a-dc5356e9ec14|ing|http://localhost:8000/examplerepository/rest/5|2018-09-19 05:23:20|SHA-1|450ddec8dd206c2e2ab1aeeaa90e85e51753b8b7|suc
[.. 20 rows total..]
sqlite> 
```

## Usage

First, start the web server by running `php bin/console server:start`. Then, run the `app:riprap:check_fixity` command, e.g., `php [path to riprap]/bin/console app:riprap:check_fixity`. If you repeat the SQL query above, you will see five more events in your database.

### REST API

Preliminary scaffolding is in place for a simple HTTP REST API, which will allow external applications like Drupal to retrieve fixity validation data on particular Fedora resources and to add new and updated fixity validation data. For example, a `GET` request to:

`curl -v -H "Resource:http://example.com/examplerepository/rest/17" http://localhost:8000/api/resource`

would return a list of all fixity events for the Fedora resource `http://example.com/examplerepository/rest/17`.

To see the API in action,

1. run `php bin/console server:start`
1. run `curl -v -H "Resource:http://example.com/examplerepository/rest/17" http://localhost:8000/api/resource`

You should get a response like this:

```
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> GET /api/resource HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.58.0
> Accept: */*
> 
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Host: localhost:8000
< Date: Fri, 07 Sep 2018 07:01:01 -0700
< Connection: close
< X-Powered-By: PHP/7.2.7-0ubuntu0.18.04.2
< Cache-Control: no-cache, private
< Date: Fri, 07 Sep 2018 14:01:01 GMT
< Content-Type: application/json
< 
* Closing connection 0
["fixity event 1 for resource http:\/\/example.com\/examplerepository\/rest\/17","fixity event 2 for resource http:\/\/example.com\/examplerepository\/rest\/17","fixity event 3 for resource http:\/\/example.com\/examplerepository\/rest\/17"]
```

HTTP `POST` and `PATCH` are also supported, e.g.:

```
curl -v -X POST -H "Resource:http://example.com/examplerepository/rest/17" http://localhost:8000/api/resource
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> POST /api/resource HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.58.0
> Accept: */*
> Resource:http://example.com/examplerepository/rest/17
> 
< HTTP/1.1 200 OK
< Host: localhost:8000
< Date: Thu, 27 Sep 2018 11:56:02 -0700
< Connection: close
< X-Powered-By: PHP/7.2.10-0ubuntu0.18.04.1
< Cache-Control: no-cache, private
< Date: Thu, 27 Sep 2018 18:56:02 GMT
< Content-Type: application/json
< 
* Closing connection 0
["new fixity event for resource http:\/\/example.com\/examplerepository\/rest\/17"]
```

### Mock Fedora repository endpoint

To assist in development and testing, Riprap includes an endpoint that simulates the behaviour described in section [7.2](https://fcrepo.github.io/fcrepo-specification/#persistence-fixity) of the spec. If you start Symfony's test server as described above, this endpoint is available via `GET` or `HEAD` requests at `http://localhost:8000/examplerepository/rest/{id}`, where `{id}` is a number from 1-20 (these are mock "resource IDs" included in the sample data). Calls to it should include a `Want-Digest` header with the value `SHA-1`, e.g.:

`curl -v -X HEAD -H 'Want-Digest: SHA-1' http://localhost:8000/examplerepository/rest/2`

If the `{id}` is valid, the response will contain the `Digest` header containing the specified SHA-1 hash:

```
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> HEAD /examplerepository/rest/2 HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.58.0
> Accept: */*
> Want-Digest: SHA-1
> 
< HTTP/1.1 200 OK
< Host: localhost:8000
< Date: Thu, 20 Sep 2018 05:28:57 -0700
< Connection: close
< X-Powered-By: PHP/7.2.7-0ubuntu0.18.04.2
< Cache-Control: no-cache, private
< Date: Thu, 20 Sep 2018 12:28:57 GMT
< Digest: b1d5781111d84f7b3fe45a0852e59758cd7a87e5
< Content-Type: text/html; charset=UTF-8
< 
* Closing connection 0
```

If the resource is not found, the response will be `404`. If the `{id}` is not valid for some other reason, the HTTP response will be `400`.

## More about Riprap

### Plugins

One of Riprap's principle design requirements is flexibility. To meet this goal, it uses plugins to process most of its input and output. It supports plugins that:

* Fetch a set of Fedora resource URLs to fixity check (e.g., from the Fedora repository's triplestore, from Drupal, from a CSV file). A sample plugin that reads resource URLs from a text file, `app:riprap:plugin:fetch:from:file`, already exists and is configured in `config/services.yaml`.
* Query an external utility or service to get the digest of the current resource. A plugin that queries a Fedora API Specification-compliant repository, `app:riprap:plugin:fetchdigest:from:fedoraapi`, and is configured in `config/services.yaml`.
* Persist data (e.g., to a RDBMS, to the Fedora repository, etc.) after performing a fixity check on each Fedora resource. A plugin to persist fixity events to a relational database, `app:riprap:plugin:persist:to:database`, already exists and is configured in `config/services.yaml`.
* Execute after performing a fixity check on each Fedora resource. Two plugins of this type are available: a plugin that sends an email on failure, `app:riprap:plugin:postvalidate:mailfailures`, and a (not yet complete) plugin that will be able to migrate fixity events from a legacy system (in this case, Fedora 3.x AUDIT data). Both plugins are confiured in `config/services.yaml`.

### Message queue listener

Riprap will also be able to listen to an ActiveMQ queue and generate corresponding fixity events for newly added or updated resources. Not implemented yet.

### Security

* Riprap requests fixity digests from other applications via HTTP or some other mechanism. For example, if Riprap is used with a Fedora-based repository, it needs access to the repository's REST interface in order to request resources' digests.
* Riprap also provides a REST interface so other applications can query it. Using Symfony's firewall to provide IP-based access to the API should provide sufficient security.

## Miscellaneous

### Running tests

From within the `riprap` directory, run:

* `php bin/phpunit`

### Maintainer

Mark Jordan (https://github.com/mjordan)

### License

[MIT](https://opensource.org/licenses/MIT)
