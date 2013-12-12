## Project layout

registry/ is the JSON API for the package registry.

www/ is the code for search.npmjs.org, eventually maybe www.npmjs.org

## Installing

**1)** Install CouchDB, version 1.1.0 or later.

**2)** Update your CouchDB local.ini and make sure the following options are set:

```
[httpd]
secure_rewrites = false
[ssl]
verify_ssl_certificates = false
ssl_certificate_max_depth = 1
```

You will need to restart CouchDB after changing these options.

**3)** Create a new database:

```
curl -X PUT http://localhost:5984/registry
```

You should receive the following answer from CouchDB:

```
{"ok":true}
```

**4)** Clone the *npmjs.org* repository if you haven't already, and cd into it:

```
git clone https://github.com/isaacs/npmjs.org.git
cd npmjs.org
```

**5)** Install the package dependencies; couchapp, semver and jsontool:

```
npm install
```

**5)** Set your internal `npm_package_config_couch` variable so it's available to `npm run` scripts:

```
npm config set _npmjs.org:couch=http://localhost:5984/registry
```

If you have set CouchDB up with an admin login you will need to change the URLs `http://user:pass@localhost:5984/registry`.

**7)** Sync the registry scratch design docs:

```
npm start
```

**8)** Load the views from CouchDB to make them active:

```
npm run load
```

**9)** Copy the scratch design docs to the active *app* to make them live:

```
npm run copy
```

## Synchronizing with the public npm registry

CouchDB now includes a *_replicator* database that is more robust than the old replication mechanism. Full details about how this works can be found here: https://gist.github.com/fdmanana/832610

To start replication, simply create a new replication task in the *_replicator* database:

```
curl -X PUT -H "Content-Type:application/json" \
    http://localhost:5984/_replicator/npm -d \
    '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry"}'
```

This replication task will restart when your server is restarted.

### Note about replication failures

CouchDB replication can fail for a number of reasons. Most commonly, timeouts in fetching data from the public registry can cause replication to halt. CouchDB will retry a number of times but if unsuccessful it won't continue to replicate. The easiest way to restart replication is to restart the CouchDB server but you can also edit the the *_replicator/npm* document and remove the `"replicationstate"` field to restart replication without restarting CouchDB.

Consider increasing timeouts and retry maximums in the CouchDB configuration files to minimise failures caused by transient network problems.

## Using the registry with the npm client

You can point the npm client at the registry by putting this in your ~/.npmrc file:

```
registry = http://localhost:5984/registry/_design/app/_rewrite
```

You can also set the npm registry config property like:

```
npm config set registry http://localhost:5984/registry/_design/app/_rewrite
```

Or you can simple override the registry config on each call:

```
npm --registry http://localhost:5984/registry/_design/app/_rewrite install <package>
```

Consider using [npmrc](https://github.com/deoxxa/npmrc) for easy *.npmrc* switching when using multiple registries.

## Optional: top-of-host urls

To be snazzier, add a vhost config:

```
[vhosts]
registry.mydomain.com:5984 = /registry/_design/app/_rewrite
search.mydomain.com:5984 = /registry/_design/ui/_rewrite
```

Where `registry.mydomain.com` and `search.mydomain.com` are
the hostnames where you're running the thing, and `5984` is the
port that CouchDB is running on. If you're running on port 80,
then omit the port altogether.

Then for example you can reference the repository like so:

```
npm config set registry http://registry.mydomain.com:5984
```

## API

### GET /packagename

Returns the JSON document for this package. Includes all known dists
and metadata. Example:

    {
      "name": "foo",
      "dist-tags": { "latest": "0.1.2" },
      "_id": "foo",
      "versions": {
        "0.1.2": {
          "name": "foo",
          "_id": "foo",
          "version": "0.1.2",
          "dist": { "tarball": "http:\/\/domain.com\/0.1.tgz" },
          "description": "A fake package"
        }
      },
      "description": "A fake package."
    }

### GET /packagename/0.1.2

Returns the JSON object for a specified release. Example:

    {
      "name": "foo",
      "_id": "foo",
      "version": "0.1.2",
      "dist": { "tarball": "http:\/\/domain.com\/0.1.tgz" },
      "description": "A fake package"
    }

### GET /packagename/latest

Returns the JSON object for the specified tag.

    {
      "name": "foo",
      "_id": "foo",
      "version": "0.1.2",
      "dist": { "tarball": "http:\/\/domain.com\/0.1.tgz" },
      "description": "A fake package"
    }

### PUT /packagename

Create or update the entire package info.

MUST include the JSON body of the entire document. Must have
`content-type:application/json`.

If updating this must include the latest _rev.

This method can also remove previous versions and distributions if necessary.

### PUT /packagename/0.1.2

Create a new release version. 

MUST include all the metadata from package.json along with dist information
as the JSON body of the request. MUST have `content-type:application/json`

### PUT /packagename/latest

Link a distribution tag (ie. "latest") to a specific version string.

MUST be a JSON string as the body. Example:

    "0.1.2"

Must have `content-type:application/json`.
