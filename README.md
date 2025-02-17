# influx-client

InfluxDB client compatible with 1.5. This client uses the awesome
[requests](http://docs.python-requests.org/en/master/) library to provide
connection pooling for each unique InfluxDB URL given.

This InfluxDB client is created, maintained, and supported by [Axiom
Exergy](http://www.axiomexergy.com).

## Prerequisites

This client has only been tested and used against InfluxDB 1.5 and Python 3.5.
If you want to support any other environments, please submit a pull request.

## Installation

You can install this client [via PyPI](https://pypi.org/project/influx-client):

```bash
$ pip install influx-client
```

Or by cloning this repository:

```bash
$ git clone https://github.com/AxiomExergy/influx-client.git
$ cd influx-client
$ pip install .  # For a regular install
$ python setup.py develop  # OR for a development install
```

## Usage

This section describes basic usage.

#### Quickstart Example

This InfluxDB client is designed to be very simple. It takes a URL to the
InfluxDB API on creation, and otherwise supplies all parameters per `write()`
call.

If the InfluxDB API returns an error that the chosen database does not exist,
the client will issue a `CREATE DATABASE ...` query, followed by retrying the
write request.

```python
from influx import InfluxDB

# This creates the client instance... subsequent calls with the same URL will
# return the exact same instance, allowing you to use socket pooling for faster
# requests with less resources.
client = InfluxDB('http://127.0.0.1:8086')

# Creating the database is optional - calls to write() will try to create the
# database if it does not exist.
client.create_database('mydatabase')

# You can write as many fields and tags as you like, or override the *time* for
# the data points
client.write('mydatabase', 'mymeasurement', fields={'value': 1.0},
             tags={'env': 'example'})

# You can write multiple datapoints at a time
client.write_many('mydatabase', 'mymeasurement', fields=['value', 'alpha'],
                  values=[[1.0, 0.5], [1.1, 0.6]], tags={'env': 'example'})

# You can query for data relative to now()
data = client.select_recent('mydatabase', 'mymeasurement', time_relative='1h')

# You can query with arbitrary WHERE clauses and LIMITs
data = client.select_where('mydatabase', 'mymeasurement', where='time > 0', limit=1)

# You can clean up after yourself, for example in testing environments
client.drop_measurement('mymeasurement', 'mydatabase')

# You can also drop the entire database, if necessary
client.drop_database('mydatabase')

# Subsequent client creation will give the same instance
client2 = InfluxDB('http://127.0.0.1:8086')
client is client2  # This is True
```

## Development

This section describes development and contribution for *influx-client*.

- Development is [on GitHub](https://github.com/AxiomExergy/influx-client).
- Installation is [via PyPI](https://pypi.org/project/influx-client).
- Issues are [on Github](https://github.com/AxiomExergy/influx-client/issues).
- Releases are [on
  GitHub](https://github.com/AxiomExergy/influx-client/releases).

### Contributors

This section lists everyone who has contributed to this project.

- [shakefu](https://github.com/shakefu) (*Creator, Maintainer*)

### Repository Layout

There are a few important pieces in this repository:

- `influx/` - The influx Python package
- `test/` - Python nosetests
- `Dockerfile`, `docker-compose.yml` - Docker configuration for testing
- `LICENSE`, `README.md` - Documentation and legal

### Running Tests

You can run the full test suite with supporting InfluxDB instance using
*docker-compose*.

The following command will build the test image and run all tests:

```bash
docker-compose up --build --force-recreate --remove-orphans --exit-code-from influx
```

When tests are complete, you can clean up supporting services using:

```bash
docker-compose down
```

### Making Pull Requests

Pull requests must pass CI to be considered for inclusion. If your pull request
does not have squashed commits, your commits should follow the *topic:
description* style. See the commit history for examples.

## API

This section describes the public API for *influx-client*.

### `influx.client(`*`url, timeout=60, precision='u'`*`)`

Helper method to allow you to instantiate an InfluxDB client directly from the
top level package.

- **url** (*str*) - URL to InfluxDB API (*required*)
- **timeout** (*int*, default `60`) - Timeout in seconds for requests
- **precision** (*str*, default `'u'`) - Precision string to use for querying

### `InfluxDB(`*`url, timeout=60, precision='u'`*`)`

This is the main InfluxDB client. It works as a singleton instance per *url*.
In threaded or event loop based environments it relies on the *requests*
library connection pooling (which in turn relies on *urllib3*) for thread
safety.

- **url** (*str*) - URL to InfluxDB API (such as `'http://127.0.0.1:8086'`)
- **timeout** (*int*, default `60`) - Timeout in seconds for requests
- **precision** (*str*, default `'u'`) - Precision string to use for querying
    InfluxDB. ([See the documentation]
    (https://docs.influxdata.com/influxdb/v1.5/tools/api/#query) for what is
    available.)

#### `.create_database(`*`database`*`)`

Issues a `CREATE DATABASE ...` request to the InfluxDB API. This is an
idempotent operation.

- **database** (*str*) - Database name

#### `.drop_database(`*`database`*`)`

Issues a `DROP DATABASE ...` request to the InfluxDB API. This will raise a 404
HTTPError if the database does not exist.

- **database** (*str*) - Database name

#### `.drop_measurement(`*`measurement, database`*`)`

Issues a `DROP MEASUREMENT ...` request to the InfluxDB API for the specified database.

- **measurement** (*str*) - Measurement name
- **database** (*str*) - Database in which *measurement* resides

#### `.write(`*`database, measurement, fields, tags={}, time=None`*`)`

Write data points to the specified *database* and *measurement*.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name
- **fields** (*dict*) - Dictionary of *field_name: value* data points
- **tags** (*dict*, optional) - Dictionary of *tag_name: value* tags to
  associate with the data points
- **time** (*datetime*, optional) - Datetime to use instead of InfluxDB's
  server-side "now"

#### `.write_many(`*`database, measurement, fields, values, tags={}, time_field=None`*`)`

Write data points to the specified *database* and *measurement*.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name
- **fields** (*list*) - List of field names, ordered the same as *values*
- **values** (*list*) - List of values (list of lists)
- **tags** (*dict*, optional) - Dictionary of *tag_name: value* tags to
  associate with the data points
- **time_field** (*str*, optional) - Field name to extract and use as timestamp

#### `.select_recent(`*`database, measurement, fields='*', tags={}, relative_time='15m'`*`)`

Query the InfluxDB API for *measurement* in *database*, using the *fields*
string, limited to matching *tags* for the recent *relative_time*.

Returns the raw JSON response from InfluxDB.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name
- **fields** (*str*, default `'*'`) - String formatted fields for `SELECT`
  query
- **tags** (*dict*, optional) - Dictionary of *tag_name: value* tags to match
- **relative_time** (*str*, default `'15m'`) - Relative time string

#### `.select_where(`*`database, measurement, fields='*', tags={}, where='time > now() - 15m', desc=False, limit=None`*`)`

Query the InfluxDB API for *measurement* in *database*, using the *fields*
string, limited to matching *tags* with the *where* clause and *limit* applied.

Returns the raw JSON response from InfluxDB.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name
- **fields** (*str*, default `'*'`) - String formatted fields for `SELECT`
  query
- **tags** (*dict*, optional) - Dictionary of *tag_name: value* tags to match
- **where** (*str*, default `'time > now() - 15m'`) Where clause to add
- **desc** (*bool*, default `False`) Add the `ORDER BY time DESC` clause
- **limit** (*int*, optional) Limit to this number of data points

#### `.select_into(`*`[database,] target, source, fields='*', where=None, group_by='*'`*`)`

Returns count of data points moved by a SELECT ... INTO ... FROM ... query.

The query will follow the format:

    SELECT *fields* INTO *target* FROM *source* WHERE *where* GROUP BY *group_by*

The WHERE and GROUP BY clauses are optional.

Note that if you do not include GROUP BY * or explicitly call out tags,
tags will be recorded as fields.

- **database** (*str*) Database name (optional)
- **target** (*str*) Target measurement
- **source** (*str*) Source measurement
- **fields** (*str*) Fields portion of the SELECT clause (optional,
    default: '*')
- **where** (*str*) WHERE portion of the SELECT clause (optional)
- **group_by** (*str*) GROUP BY portion of the SELECT clause (optional,
    default: '*')

This method may be called with variable arguments, if you wish to specify the
full qualified measurement name with database.

Example:

```python
InfluxDB().select_into('database', 'target', 'source')
# Is the same as
InfluxDB().select_into('database.default.target', 'database.default.source')
# Where often the default is 'autogen'
```

#### `.show_tags(`*`database, measurement`*`)`

Query the InfluxDB API and return a list of tag names in *database* and
*measurement*.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name

#### `.show_fields(`*`database, measurement`*`)`

Query the InfluxDB API and return a list of field names in *database* and
*measurement*.

- **database** (*str*) - Database name
- **measurement** (*str*) - Measurement name

## License

This repository and its codebase are made public under the [Apache License
v2.0](./LICENSE). We ask that if you do use this work please attribute [Axiom
Exergy](http://www.axiomexergy.com) and link to the original repository.

## Changelog

See [Releases](https://github.com/AxiomExergy/influx-client/releases) for
detailed release notes.
