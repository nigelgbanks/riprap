![RipRap](https://user-images.githubusercontent.com/2371345/48165629-86513c80-e2bc-11e8-9577-dc0525c74184.png)
# Riprap

[![Contribution Guidelines](http://img.shields.io/badge/CONTRIBUTING-Guidelines-blue.svg)](./docs/CONTRIBUTING.md)
[![LICENSE](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](./LICENSE)

A PREMIS-compliant fixity checking microservice. Developed as a successor to Islandora 7.x's [Checksum Checker](https://github.com/Islandora/islandora_checksum_checker) module (see [this Github issue](https://github.com/Islandora-CLAW/CLAW/issues/847) for background), it is intended primarily to be used with repositories compliant with the [Fedora API Specification](https://fedora.info/spec/), but can be used to provide fixity validation for other repositories as well (e.g., an [OCFL](https://ocfl.io/) repository). In fact, Riprap ships with sample plugins that allow it to monitor the fixity of files on a standard attached filesystem and call `sha1sum` to get their current digests.

Riprap periodcally requests fixity digests for resources from a repository and compares the digest with a previously request digest. It then persists the outocome of that comparison so the process can be repeated again. Riprap also provides a REST interface so that external applications (in Islandora's case, Drupal) can retrieve fixity checking event data for use in reports, etc.

![Overview](docs/images/overview.png)

Riprap generates and records fixity check events as described in the "Fixity, integrity, authenticity" section of the [PREMIS Data Dictionary for Preservation Metadata, Version 3.0](https://www.loc.gov/standards/premis/v3/premis-3-0-final.pdf). It can also record fixity information available during "ingestion" events and available at the time of "deletion" events. "fixity check" events are generated by Riprap, typically in a job scheduled via `cron`, but "ingestion" and "deletion" events are generated by external systems, which may persist information about those events at any time in Riprap's database via either its REST API or via an ActiveMQ message. Initial fixity checks (that is, after a resource is ingested into the repository) and final fixity checks (just before a resource is deleted from the repository) can be identified by adding a brief note to the event.

All events must have a value of `success` or `fail`, using values from the Library of Congress' Preservation [Event Outcome](http://id.loc.gov/vocabulary/preservation/eventOutcome.html) vocabulary.

## Current status

Riprap's major functionality is in place, with the exception of the ActiveMQ event queue listener and write operations (`POST` and `PATCH`) in the REST API.

Various combinations of Riprap's current fixity auditing capabilities are illustrated in "The sample configuration files" section below. Additional funcitonality can be added via new plugins (contibutions are welcome).

## Requirements

* PHP 7.1.3 or higher
* [composer](https://getcomposer.org/)
* An SQLite, MySQL, or PostgreSQL relational database, with appropriate PHP drivers.

While not a requirement, a [module for Islandora](https://github.com/mjordan/islandora_riprap) is available that provides node-level fixity reports on binary resources using data from Riprap. Similar reporting tools could be developed for other platforms.

## Installation

1. Clone this git repository
1. `cd riprap`
1. `php composer.phar install` (or equivalent on your system, e.g., `./composer install`)
1. If necessary, create the database as described [here](docs/databases.md).

We will eventually support deployment via Ansible.

## Trying it out

If you want to play with Riprap, and you're on a Linux or OSX machine, you should not need to configure anything. Riprap comes with three sample configuration files that are ready to use (we will describe each one below). You do not need to create a database to try out Riprap using the "filesystemexample" configuration, but if you want to use the "mockfedorarepository" or "islandora" configurations described below, or create your own plugins that use a database, you will need to create a database using [these instructions](docs/databases.md).

## The sample configuration files

Riprap comes with three sample configuration files:

* `services.yaml.filesystemexample`: This configuration checks the fixity of the files in a specified directory.
* `services.yaml.mockfedorarepository`: This configuration checks the fixity of a set of resources in a mock Fedora API-compliant repository. Riprap includes this mock endpoint via its built-in web server.
* `services.yaml.islandora`: This configuration is used in conjuction with an Islandora 8.x-1.x instance, such as the one provided by the [CLAW Vagrant Playbook](https://github.com/Islandora-Devops/claw-playbook). It audits the fixity of resources in a real (not mock) Fedora 5 repository.

To use these configuration files, copy the one you want to try from `config/[filename]` to `config/services.yaml`, which is the configuration file that Riprap uses when you run the `check_fixity` command.

### The Filesystemexample configuration

Whereas the other two sample configurations audit the fixity of resources in a Fedora API-compliant repository (either a mocked up one or a real one) and perist fixity events to a relational database, the "filesystemexample" configuration audits a set of files in a filesystem directory and perists events to a CSV file. While you could use this configuration in production, its real purpose is to illustrate how Riprap plugins work together to provide all the functionality required to audit fixity over time.

Let's look at the `config/services.yaml.filesystemexample` configuration file to see what the plugins are doing. The section we are interested in (with line number added so we can refer to specific lines in the explanation) is:

```
 ### 'filesystemexample' plugins
 1. app.plugins.fetchresourcelist: ['app:riprap:plugin:fetchresourcelist:from:glob']
 2. app.plugins.fetchresourcelist.from.glob.file_directory: 'resources/filesystemexample/resourcefiles'
 3. app.plugins.fetchdigest: 'app:riprap:plugin:fetchdigest:from:shell'
 4. app.fixity_algorithm: 'SHA-1'
 5. app.plugins.fetchdigrest.from.shell.command: '/usr/bin/sha1sum'
 6. app.plugins.persist: ['app:riprap:plugin:persist:to:csv']
 7. app.plugins.persist.to.csv.output_path: '%kernel.project_dir%/var/riprap_persist_to_csv_plugin_events.csv'
```

The "fetchresourcelist" plugin registered in line 1 corresponds to the plugin class file located at `src/Command/PluginFetchResourceListFromGlob.php`. If you look at that file, it's pretty simple - its `execute()` function returns a file path for each of the files ending in `.bin` in a directory.

The specific directory that plugin lists files from is indicated in line 2. In this case, the directory is relative to the riprap installation directory (e.g., `resources/`).

Whereas the "fetchresourcelist" plugin provides a list of resources (files) whose fixity we want to audit, the "fetchdigest" plugin (identified in line 3) generates the digest that we use in the audit. In this case, that plugin is `app:riprap:plugin:fetchdigest:from:shell`, which corresponds to the class file at `src/Command/PluginFetchDigestFromShell.php`. If you look at that class'es `execute()` function, you will see that if runs the command identified in the `app.plugins.fetchdigrest.from.shell.command` configuration option (line 5) on each of the files listed by the "fetchresourcelist" plugin. All configurations must register the digest algorithm (line 4) they are using.

There is a third plugin that Riprap requires to do its job: a plugin that persists the outcome of the fixity check event somewhere. This class of plugin is registered in the `app.plugins.persist` option (line 6). In this case, that plugin corresponds to the PHP file at `src/Command/PluginPersistToCsv.php`. The path to the CSV file is identified in line 7. `%kernel.project_dir%` is Symfony's environment variable containing the path to the running project, or in our case, the directory `riprap` is located in. Each of the CSV records in this file corrsponds to a fixity check event on a specific file; Riprap uses the last event to confirm that the digest (SHA-1, for example) it generates during execution is identical. If it is, the event is successful; if it is not, the event is flagged as a failre.

If you copy `config/services.yaml.filesystemexample` to `config/services.yaml` and run Riprap:

`php bin/console app:riprap:check_fixity`

you will see the persisted events in the CSV file at `var/riprap_persist_to_csv_plugin_events.csv`, one per `.bin` file under the `resources/ ` directory. If you rerun Riprap, you will see three more events in the CVS file.

This walkthrough of the "filesystem" plugins illustrates Riprap's basic functionality: it takes a list of resources (a.k.a. files) whose fixity we are auditing, and for each of those resources, gets the digest using a particular hash_algorithm. Riprap then checks the current digest against the digest of the same algorithm in the previous fixity check event for the resource, and finally saves the outcome of the current fixity check event for use in the next execution cycle.

This walkthrough also illustrates how a developer would write additional plugins for performing fixity auditing of resources managed by arbitrary storage platforms.

### The Mock Fedora Repository configuration

Before we describe how to use Riprap against the mock Fedora endpoint, we should tell you a little about it. As its name suggests, the endpoint simulates the behaviour described in section [7.2](https://fcrepo.github.io/fcrepo-specification/#persistence-fixity) of the Fedora API spec. If you start Symfony's test server as described below, this endpoint is available via `GET` or `HEAD` requests at `http://localhost:8000/mockrepository/rest/{id}`, where `{id}` is a number from 1-20 (these are mock "resource IDs" included in the sample data). Calls to it should include a `Want-Digest` header with the value `SHA-1`, for example:

`curl -v -X HEAD -H 'Want-Digest: SHA-1' http://localhost:8000/mockrepository/rest/2`

If the `{id}` is valid, the response will contain the `Digest` header containing the specified SHA-1 hash:

```
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> HEAD /mockrepository/rest/2 HTTP/1.1
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

To start the web server, from within the `riprap` directory run the following command:

`php bin/console server:start`

This will start the server listening on `localhost:8000`. The [official Symfony documentation](https://symfony.com/doc/current/setup/built_in_web_server.html) provides details on how to start it on a different port.

After the web server starts, run Riprap:

`php bin/console app:riprap:check_fixity`

You should see output similar to:

`Riprap checked 5 resources (5 successful events, 0 failed events) in 0.223 seconds.`

Here is what is going on when you run the `check_fixity` command against the mock Fedora repository endpoint:

1. Riprap calls whatever `fetchresourcelist` plugins are enabled, and from them gets a list of all resources to check. In the default sample configuration, this list of resources is a plain text file at `resources/iprap_resource_ids.txt`.
1. For each of the resources identifed by the `fetchresourcelist` plugins, Riprap calls the `fetchdigest` plugin that is enabled, and gets the resource's digest value from the repository. In the default sample configuration, Riprap is calling its mock repository endpoint.
1. Riprap then gets the digest value in the most recent fixity check event stored in its database (in the default sample configuration, this is fixity events stored in the relational database), and compares the newly retrieved digest value with the most recent one on record.
1. Riprap then persists information about the fixity check event it just performed (in the default sample configuration, into a  database table).
1. Riprap then executes all `postcheck` plugins that are enabled (see the "Plugins" section below for more info on this class of plugin).
1. After Riprap has checked all resources in the current list, it reports out how many resources it checked, including how many checks were successful and how many failed.

If you query Riprap's database table you will see the fixity events. From within the `riprap` directory, run `sqlite3 var/data.db` to access the database. then run the commands illustrated below (this example uses SQLite, but the `SELECT` statement works in all relational databases):

```
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .headers on
sqlite> select * from fixity_check_event;
id|event_uuid|event_type|resource_id|timestamp|digest_algorithm|digest_value|event_detail|event_outcome|event_outcome_detail_note
1|92224d93-563a-4c6e-8a2e-251084fb9cdc|fix|http://localhost:8000/mockrepository/rest/10|2018-10-01 07:49:13|SHA-1|c28097ad29ab61bfec58d9b4de53bcdec687872e|Initial fixity check.|success|
2|4e0efd4e-f6c5-4e7d-af4c-015b696f6047|fix|http://localhost:8000/mockrepository/rest/11|2018-10-01 07:49:13|SHA-1|339e2ebc99d2a81e7786a466b5cbb9f8b3b81377|Initial fixity check.|success|
3|70c91ae4-6a3e-4160-8985-00b4ffc626f7|fix|http://localhost:8000/mockrepository/rest/12|2018-10-01 07:49:13|SHA-1|0bad865a02d82f4970687ffe1b80822b76cc0626|Initial fixity check.|success|
4|10af3a5f-309a-4962-9b80-a6f8c17d8a0c|fix|http://localhost:8000/mockrepository/rest/13|2018-10-01 07:49:13|SHA-1|667be543b02294b7624119adc3a725473df39885|Initial fixity check.|success|
5|e64db74c-471e-4347-b256-5597470157c4|fix|http://localhost:8000/mockrepository/rest/14|2018-10-01 07:49:13|SHA-1|86cf294a07a8aa25f6a2d82a8938f707a2d80ac3|Initial fixity check.|success|
sqlite>
sqlite> .quit
```

If you rerun the `check_fixity` command, and then repeat the SQL query above, you will see five more events in your database, one corresponding to each URL listed in `resources/iprap_resource_ids.txt`.

(Note that if you populated the database with sample data prior to running `check_fixity`, you will also see those 20 events in the database.)

### The Islandora configuration

> If you are running Islandora in a CLAW Playbook Vagrant guest virtual machine and Riprap on the Vagrant host machine, start the Riprap web server by running `php bin/console server:start *:8001` in the Riprap directory. See the [Islandora Riprap](https://github.com/mjordan/islandora_riprap) README file for more information. Otherwise, the Symfony web server will have a port conflict with the Apache web server mapped to port `8000` on the host machine.

The "islandora" configuration works like the other two sample configurations, but it queries Drupal's JSON:API for the list of resources to audit (using the descriptively named `app:riprap:plugin:fetchresourcelist:from:drupal` plugin), and it queries the REST API of the (real, not mock) Fedora repository that accompanies Drupal in the Islandora stack for the digests of those files (using the `app:riprap:plugin:fetchdigest:from:fedoraapi` plugin).

Within the Drupal user interface, the [Islandora Riprap](https://github.com/mjordan/islandora_riprap) module provides reports on whether Riprap has recorded any failed fixity check events (i.e., digest mismatches for the same resource) over time. The module gets this information via the Riprap REST API, described in the next section.

## Riprap's REST API

Riprap provides a simple HTTP REST API (completely separate from the mock Fedora API), which will allow external applications like Drupal to retrieve fixity check data on specific Fedora resources and to add new and updated fixity check data. For example, a `GET` request to:

`curl -v -H "Resource-ID:http://example.com/repository/resource/12345" http://localhost:8000/api/fixity`

would return a list of all fixity events for the Fedora resource `http://example.com/repository/resource/12345`.

To see the API in action,

1. run `php bin/console server:start`
1. run `curl -v -H 'Resource-ID:http://localhost:8000/mockrepository/rest/10' http://localhost:8000/api/fixity`

You should get a response like this:

```
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> GET /api/fixity HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.58.0
> Accept: */*
> Resource-ID:http://localhost:8000/mockrepository/rest/10
>
< HTTP/1.1 200 OK
< Host: localhost:8000
< Date: Sun, 30 Sep 2018 10:13:49 -0700
< Connection: close
< X-Powered-By: PHP/7.2.10-0ubuntu0.18.04.1
< Cache-Control: no-cache, private
< Date: Sun, 30 Sep 2018 17:13:49 GMT
< Content-Type: application/json
<
```

The returned JSON looks like this:

```javascript
[
   {
      "event_uuid":"4cd2edc9-f292-49a1-9b05-d025684de559",
      "resource_id":"http:\/\/localhost:8000\/mockrepository\/rest\/10",
      "event_type":"fix",
      "timestamp":"2018-10-03T07:23:40-07:00",
      "hash_algorithm":"SHA-1",
      "hash_value":"c28097ad29ab61bfec58d9b4de53bcdec687872e",
      "event_detail":"Initial fixity check.",
      "event_outcome":"suc",
      "event_outcome_detail_note":""
   },
   {
      "event_uuid":"fb73a36a-df64-4ba8-a437-ea277b65ebb7",
      "resource_id":"http:\/\/localhost:8000\/mockrepository\/rest\/10",
      "event_type":"fix",
      "timestamp":"2018-12-03T07:26:39-07:00",
      "hash_algorithm":"SHA-1",
      "hash_value":"c28097ad29ab61bfec58d9b4de53bcdec687872e",
      "event_detail":"",
      "event_outcome":"suc",
      "event_outcome_detail_note":""
   }
   [...]
]
```
Note that if the resource identified by `Resource-ID` does not have any events in Riprap, the REST API will return a `200` response and an empty body, e.g.,

```javascript
[]
```

This means that consumers of this API will need to not only check for the HTTP response code, but also count the number of members in the returned list.

HTTP `POST` and `PATCH` will also be supported, e.g.:

```
curl -v -X POST -H "Resource-ID:http://localhost:8080/mockrepository/rest/17" http://localhost:8000/api/fixity
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> POST /api/fixity HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.58.0
> Accept: */*
> Resource-ID:http://localhost:8080/mockrepository/rest/17
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
["new fixity event for resource http:\/\/localhost:8080\/mockrepository\/rest\/17"]
```

`GET` requests can optionally take the following URL parameters:

* `timestamp_start`: ISO8601 (full or partial) date indicating start of date range in queries.
* `timestamp_end`: ISO8601 (full or partial) date indicating end of date range in queries.
* `outcome`: Coded outcome of the event, either `success` or `fail`.
* `offset`: The number of items in the result set, starting at the beginning, that are skipped in the result set (i.e., same as standard SQL use of 'offset'). Default is 0.
* `limit`: Number of items in the result set to return, starting at the value of `offset`.
* `sort`: Sort events on timestamp. Specify "desc" or "asc" (if not present, will sort "asc").

For example, `curl -v -H 'Resource-ID:http://localhost:8000/mockrepository/rest/10' http://localhost:8000/api/fixity?timestamp_start=2018-12-03` would return only the events for `http://localhost:8000/mockrepository/rest/10` that have a timestamp equal to or later than `2018-12-03`.

## More about Riprap

### Plugins

One of Riprap's principle design requirements is flexibility. To meet this goal, it uses plugins to process most of its input and output. We have already been introduced to three types (and different combinations of) plugins in the sample configurations above:

* "fetchresourcelist" plugins fetch a set of resource URIs/URLs to fixity check (e.g., from a Fedora repository's triplestore, from Drupal, from a CSV file). Multiple fetchresourcelist plugins can be configured to run at the same time.
* "fetchdigest" plugins query an external utility or service to get the digest of the current resource. Only one fetchdigest plugin can be configured.
* "persist" plugins persist data after performing a fixity check on each resource (e.g. to a RDBMS, back into the Fedora repository that manages the resources, etc.). Multiple persist plugins can be configured to run at the same time..

Riprap supports a fourth class of plugin, which we didn't see in our sample configurations:

* "postcheck" plugins execute after performing a fixity check on each resource. Multiple postcheck plugins can be configured to run at the same time. Preliminary versions of two plugins of this type currently exist (but neither one is complete yet): a plugin that sends an email on failure, `app:riprap:plugin:postcheck:mailfailures`, and a plugin that migrates fixity events from Fedora 3.x AUDIT data.

### Message queue listener

Riprap will also be able to listen to an ActiveMQ queue and generate corresponding fixity events for newly added or updated resources. Not implemented yet.

### Security

* Riprap retrieves fixity digests from other applications via HTTP or some other mechanism. If Riprap is used with a Fedora-based repository, it needs access to the repository's REST interface in order to request resources' digests.
* Riprap also provides a REST interface so other applications can retrieve fixity check event data from it and add/modify fixity check event data. Using Symfony's firewall to provide IP-based access to the API should provide sufficient security.

## Miscellaneous

### To do

* Add an ActiveMQ listener.
* Add write operations to the REST interface.
* Complete the postcheck plugins.
* Write more plugins so Riprap can be used with additional storage platforms.

### Contributing

See [CONTRIBUTING.md](docs/CONTRIBUTING.md). Functional and unit tests, and additional plugins, are most welcome.

### Running tests

From within the `riprap` directory, run:

`./bin/phpunit`

### Coding standards

Riprap follows the [PSR2](https://www.php-fig.org/psr/psr-2/) coding standard. To check you code, from within the `riprap` directory, run:

` ./vendor/bin/phpcs`

### Maintainer

Mark Jordan (https://github.com/mjordan)

### License

[MIT](https://opensource.org/licenses/MIT)
