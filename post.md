Consider the following, rather simplistic, [Node.js][node] module:

```javascript
var Neo4j = require('rainbird-neo4j');

function graphsAreEverywhere(url, callback) {
    var neo4j = new Neo4j(url);

    neo4j.query('CREATE (graphs)-[are]->(everywhere)', function(err) {
        callback(err);
    });
}

module.exports.graphsAreEverywhere = graphsAreEverywhere;
```

We're not doing a huge amount. We've defined a function that, when called, will attempt to run some [Cypher][cypher] on a [Neo4j][neo4j] database living on the provided URL.

Unit testing this code, while possible, doesn't do much.

```javascript
var expect = require('chai').expect;
var rewire = require('rewire');
var Neo4j = require('rainbird-neo4j');
var sample = rewire('../index.js');

describe('Our sample code', function() {

    before(function(done) {

        Neo4j.prototype.query = function(statement, callback) {
            callback(null);
        };

        sample.__set__('Neo4j', Neo4j);

        done();
    });

    it('should run the query', function(done) {
        sample.graphsAreEverywhere('url', function(err) {
            expect(err).to.not.be.ok();
            done();
        })
    });
});
```

Here I've just stubbed the Neo4j driver something that does nothing. This effectively tests that our callback works, _and nothing else_.

Despite now having 100% coverage on my code, and a passing unit test, there's a bug. My Cypher is wrong. The relationship is missing a colon. It should read:

```javascript
'CREATE (graphs)-[:are]->(everywhere)'
```

Unit tests are great, but sometimes nothing beats actually connecting to a database and testing that what we think we're running is actually being run. This can pose a problem, however.

Consider the following functional test:

```javascript
var expect = require('chai').expect;
var Neo4j = require('rainbird-neo4j');
var sample = require('../index.js');

describe('Our sample code', function() {

    var db;

    /* jshint sub: true */
    var url = process.env['NEO4J_TEST_URL'] || 'http://localhost:7474';
    /* jshint sub: false */

    before(function(done) {
        db = new Neo4j(url);
        done();
    });

    it('should write to the database', function(done) {
        sample.graphsAreEverywhere(url, function(err) {
            var query = 'MATCH (s)-[r]->(o) RETURN s, r, o';

            expect(err).to.not.be.ok();
            db.query(query, function(err, results) {
                expect(results[0]).to.have.length(1);
                done();
            });
        })
    });
});
```

This test only works the first time it's run. Second and subsequent calls result in the test failing because it's finding more data than it expects.

With this simple test I could quite easily clear down the database before I start, or make a note of how many results are returned before our function is called and simply ensure it is increased by one. In the real world this may not be feasable. For example, there is no way to _fully_ clear down a Neo4j database from within the database itself.

To ensure that the database is in a consistent state between each test run it's preferable to run a fresh instance of Neo4j each time the test is run. The tests can then populate the database as needed before testing various interactions.

The easiest way to provision a new Neo4j database for each test run is to simply run it in [Docker][docker]. In fact someone has already created a [Docker image for Neo4j][neo4jdocker].

We can now run this using a very simple `docker` command:

```bash
docker run -i -t -d --name neo4j -p 7474:7474 tpires/neo4j
```

> **Note**: Some installations may require you to run Docker using `sudo`. If this is the case use `sudo docker` rather than just `docker`.

This command is telling Docker to run the image `tpires/neo4j` on port `7474` and call the resulting Docker instance `neo4j`. The image will be downloaded from [Docker Hub][hub] and contains just what's needed to get Neo4j up and running.

We can now run our tests against the instance of Neo4j running in Docker.

> **Note:** Due to the way Docker runs on OSX you may need to use the IP address of the VirtualBox host that Docker is using rather than using localhost in your connection URL. If you run `export NEO4J_TEST_URL=http://$(boot2docker ip):7474` this will run the above test on the correct URL.

Once our tests are run we can then tear down the Noe4j instance using:

```bash
docker stop neo4j
docker rm neo4j
```

The next time we use our `docker run` command a fresh, empty instance of Neo4j will be run.

## Advanced Use

The `-p 7474:7474` part of our command tells Docker to map port `7474` from the Docker image to port `7474` on the host operating system. We can change this to map to any port on the host operating system that we like. This allows us to run test instances of Neo4j side by side, or alongside a native instance. Running:

```bash
docker run -i -t -d --name neo4j-one -p 7575:7474 tpires/neo4j
docker run -i -t -d --name neo4j-two -p 7676:7474 tpires/neo4j
```

Will spin up two instances of Neo4j, one called `neo4j-one` running on port `7575`, and one called `neo4j-two` running on port `7676`. This allows multiple functional tests to be run in parallel. 

Cleaning these instances up is just a question of stopping and deleting them:

```bash
docker stop neo4j-one neo4j-two
docker rm neo4j-one neo4j-two
```

## Sample Code

The code used in this article can be found on [GitHub][github].

[docker]: https://www.docker.com/
[node]: https://nodejs.org/
[cypher]: http://neo4j.com/developer/cypher-query-language/
[neo4j]: http://neo4j.com/
[neo4jdocker]: https://github.com/tpires/neo4j
[hub]: https://hub.docker.com
[github]: https://github.com/domdavis/functional-testing-neo4j-docker