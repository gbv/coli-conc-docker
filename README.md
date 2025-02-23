# coli-conc Docker Setup
This repository shows how to set up your own coli-conc infrastructure using Docker. [More information on the coli-conc project.](https://coli-conc.gbv.de)

We will be using:
- Cocoda (web-based tool for creating mappings) ([GitHub](https://github.com/gbv/cocoda), [Docker Info](https://github.com/gbv/cocoda/blob/dev/docker/README.md))
- JSKOS Server (a web service to access JSKOS data) ([GitHub](https://github.com/gbv/jskos-server), [Docker Info](https://github.com/gbv/jskos-server/blob/master/docker/README.md))
- Login Server (authentication server used in Cocoda to authenticate to JSKOS Server) ([GitHub](https://github.com/gbv/login-server), [Docker Info](https://github.com/gbv/login-server/blob/master/docker/README.md))
- MongoDB (database for storing our data) ([Website](https://www.mongodb.com), [Docker Hub](https://hub.docker.com/_/mongo))

See also the repository <https://github.com/gbv/coli-conc-server> for the full setup of coli-conc services,  including the services listed above, configured to be hosted at <https://coli-conc.gbv.de/>.

## Setup
Requirements: [Docker](https://docs.docker.com/engine/), [Docker Compose](https://docs.docker.com/compose/)

Note that, depending on your system, you might need to use `docker compose` with `sudo`.

1. Clone this repository:
```bash
git clone https://github.com/gbv/coli-conc-docker.git
cd coli-conc-docker
```

2. Create necessary data folders:
```bash
mkdir -p data/{jskos-server,login-server,cocoda,db}
```

3. Start the login-server container so that it can create a keypair that we will need to configure jskos-server:

```bash
docker compose up login-server
```

Stop the container using Ctrl+C as soon as it says "Listening on port 3004."

## Configuration Files

### Cocoda
Let's create a basic configuration file which sets up Cocoda to use our own instances of jskos-server (for schemes/concepts and mappings) and login-server (for authentication):

```bash
cat <<EOT > data/cocoda/cocoda.json
{
  "title": "My Cocoda",
  "auth": "http://localhost:30104",
  "registries": [
    {
      "provider": "ConceptApi",
      "uri": "http://coli-conc.gbv.de/registry/coli-conc-concepts-local",
      "status": "http://localhost:30101/status",
      "notation": [
        "J"
      ],
      "prefLabel": {
        "en": "JSKOS Server"
      }
    },
    {
      "provider": "MappingsApi",
      "uri": "http://coli-conc.gbv.de/registry/coli-conc-mappings-local",
      "status": "http://localhost:30101/status",
      "notation": [
        "C"
      ],
      "prefLabel": {
        "en": "Mappings"
      }
    }
  ],
  "overrideRegistries": true,
  "favoriteSchemes": [
    "https://www.ixtheo.de/classification/",
    "http://bartoc.org/en/node/1940"
  ]
}
EOT
```

### jskos-server
Let's create the basic configuration file for jskos-server and inject the public key that login-server created earlier into it:

```bash
cat <<EOT > data/jskos-server/config.json
{
  "baseUrl": "http://localhost:30101",
  "mongo": {
    "host": "db"
  },
  "auth": {
    "key": "$(awk '{printf "%s\\n", $0}' data/login-server/public.key  | rev | cut -c3- | rev)"
  }
}
EOT
```

jskos-server offers a variety of configuration options, especially in regards to write access. Please refer to [the documentation](https://github.com/gbv/jskos-server#configuration) on how to further configure it. For now, the default configuration (apart from the above) should be sufficient.

## Startup, Data Import, and Final Configuration
At this point, we can start the whole suite of applications using Docker Compose:

```bash
docker compose up -d
```

### Data Import
As an example, we are going to import the [IxTheo classification](https://www.ixtheo.de) into jskos-server:

```bash
# Indexes only have to created once per database
docker compose exec -it jskos-server /usr/src/app/bin/import.js --indexes
# We'll import all coli-conc related vocabularies from BARTOC, which will automatically allow us to access them via their respective APIs
docker compose exec -it jskos-server /usr/src/app/bin/import.js schemes "https://bartoc.org/api/voc?partOf=http%3A%2F%2Fbartoc.org%2Fen%2Fnode%2F18926"
# Importing the IxTheo concepts (and update IxTheo's `API` field in the local database)
docker compose exec -it jskos-server /usr/src/app/bin/import.js concepts --set-api https://raw.githubusercontent.com/gbv/jskos-data/master/ixtheo/ixtheo.ndjson
```

For importing data from local files instead of URLs, please note the caveat [on the bottom of the Docker documentation](https://github.com/gbv/jskos-server/tree/master/docker#data-import).

### Create User in Login Server
Use the included script to create a new local provider with a test user:

```bash
docker compose exec login-server /usr/src/app/bin/manage-local.js
```

Hit Ctrl+C to exit the script after you created the provider and user.

**Note:** This is mostly to have an account for testing. For productive use, you can also use one of a few available identity providers ([documentation](https://github.com/gbv/login-server#strategies)).

### Restart and Launch
We have to restart the containers one last time to apply the changes:

```bash
docker compose restart
```

Now open Cocoda at http://localhost:30100/. On the top right, hover over "Account" and click on Login. In the popover, click on "Log in via <name of your provider>" and use the just created user to log in. Close the popover. Cocoda is now ready to be used.

## Notes
- If you change any of the ports (or use a proxy), you need to adjust the configuration files of Cocoda and jskos-server as well as the entry for login-server in the `docker-compose.yml` file.
  - Examples on how to use it with a proxy will be added later.
  - We will try to simplify this by using environment variables for the base URLs and ports for all services. For now, manual adjustment is necessary.
- When changing the configuration of jskos-server or login-server, you need to restart those containers (e.g. by running `docker compose restart jskos-server`). Cocoda requests the configuration file on startup, so you only need to reload the web page to apply the changes.
- To update the containers, run `docker compose pull; docker compose up -d`. Note that it might make sense for you to pin the containers to their major version in `docker-compose.yml` to prevent issues when updating (e.g. by using `ghcr.io/gbv/cocoda:1`). If we are releasing a new major version, we will add instructions on how to upgrade your installation.

## Questions
Please create an issue in this repository for any general questions or issues related to the Docker setup. If you think you have found a specific issue related to one of the tools, please open an issue in the respective repository.
