# OCCIware demo - Linked Data server on Docker optimized for analytics

This demo showcases OCCIware Studio deploying a complete, working Ozwillo Datacore cluster (one Java and 3 mongo replica nodes) on Docker both locally and on a remote Open Stack VM, and developing a custom OCCI extension (including designer and connector) for Linked Data that allows to publish data projects and let them use a specific mongo secondary rather than the whole cluster (typically for read-heavy queries such as for analytics). This last point is achieved by visually linking OCCI Resources across Cloud layers : from Linked Data as a Service (LDaaS) to Infrastructure as a Service (IaaS).

It has been shown at EclipseCon France 2016 on June the 9th (with versions 1.0 of Docker images and 2cada878ecaf901fb7750d65b6cda66815467ff2 of Datacore).

## Prerequisites :

Java 8, Maven 3, OCCIware Studio (20160609 source), VirtualBox (5.0 ?), Docker 1.8.3 (workarounds must be applied with later versions), and the ozwillo-datacore-occiware_safe_20160630.tar.gz demo archive.

Also open the EclipseCon slides to get screenshots of all steps.

Note that there is no Ozwillo Datacore-specific software to install and deploy because it is all packaged within public Docker images.

### In finer details :

Downgrade to the right version of Docker & Docker Machine :

sudo apt-get purge docker-engine
sudo apt-get autoremove --purge docker-engine
sudo apt­-get install docker­-engine=1.8.3­-0~trusty
sudo docker version
curl -L https://github.com/docker/machine/releases/download/v0.4.1/docker-machine_linux-amd64 > a
sudo mv a /usr/local/bin/docker-machine
sudo chmod +x /usr/local/bin/docker-machine
sudo docker-machine -version
docker-machine version 0.4.1 (e2c88d6)

Downgrade to the right version of virtualbox :

sudo apt-get remove virtualbox
sudo dpkg -i Binaires\ VirtualBox/Ubuntu\ 14.04/virtualbox-5.0_5.0.18-106667-Ubuntu-trusty_amd64.deb

Build the right version of OCCIware Studio (takes 40 minutes) :

git clone git@github.com:occiware/ecore.git
git checkout fbeb291019e8dba176bcc5937aef55e7c1fe0883
cd clouddesigner
mvn clean install

### Other tips :
BEWARE docker-machine doesn't allow hypens in its VM names (better in v2)


## Run the Linked Data designer from the demo archive :

Extract the ozwillo-datacore-occiware_safe_20160630.tar.gz archive and import all its projects in the OCCIware Studio :

org.occiware.clouddesigner.occi.linkeddata
org.occiware.clouddesigner.occi.linkeddata.connector
org.occiware.clouddesigner.occi.linkeddata.connector.dependencies
org.occiware.clouddesigner.occi.linkeddata.design
org.occiware.clouddesigner.occi.linkeddata.edit
org.occiware.clouddesigner.occi.linkeddata.editor
org.occiware.clouddesigner.occi.linkeddata.tests
ozwillo-datacore-occiware

Download the Linked Data connector dependencies by doing, in the org.occiware.clouddesigner.occi.linkeddata.connector.dependencies project :
mvn clean install

Refresh the connector project. Its lib/ directory should now be filled with dependency jars.

Run the Linked Data Extension designer (Run as > Eclipse Application).

In it, import the extracted ozwillo-datacore-occiware project, and open its files:
- ozwillo-datacore-cluster.docker using the Docker designer
- Mytest.linkeddata using the Linked Data designer.


## To make the sample OCCI configurations work, on a local VirtualBox (using Boot2Docker) :

1. In the Docker Studio on said VM (ozwillodatacoredevlocal in the ozwillo-datacore-cluster.docker configuration), do the VM's "Start all" action. Wait until VM has been created and Docker images downloaded.
If Docker containers don't start, try the VM's "Synchronize" action, else each Container's "Restart" action, or start them manually (at least the mongo ones) (docker start ozwillo-mongo-1/2/3).

2. Initiate the mongo replica set with ozwillo-mongo-1 as primary :
(LATER or on Docker 1.8 their hostnames should have been set and it should be possible to use them rather than IPs)
(LATER this init should be doable by an OCCI Component at Platform level ex. MongoDatabase configuring a PaaS ex. Roboconf)
docker-machine ssh ozwillodatacoredevlocal
docker exec -it ozwillo-mongo-1 mongo
use admin
rs.initiate()
// using IP since studio can't --add-host in both ways using links
//rs.add("172.17.0.2:27017") // NO it should be itself (check in /etc/hosts) and would raise error 13433 exception: can't find self in new replset config
rs.add("172.17.0.3:27017")
rs.add("172.17.0.4:27017")
cfg = rs.conf()
// one host is wrong, depending on the order of container creation :
// error 13433 exception: can't find self in new replset config
// => replace this one's docker-gen'd host by ip https://sebastianvoss.com/docker-mongodb-sharded-cluster.html
cfg.members[0].host = "172.17.0.2:27017"
// preventing ozwillo-mongo-2 from becoming master :
// (so that it can be targeted as custom secondary by an LDDatabaseLink without risk)
// https://docs.mongodb.com/v2.6/tutorial/configure-secondary-only-replica-set-member/
cfg.members[1].priority = 0
rs.reconfig(cfg)
rs.status()
// wait until rs.status() says all other replica are SECONDARY

3. Start (or restart) ozwillo-datacore-1 container the same way.
WORKAROUND for Boot2Docker > 1.8 ex. 1.11 :
then quickly add the ozwillo-mongo-2 host definition by executing :
docker exec -it ozwillo-datacore-1 /bin/bash
echo 172.17.0.3 ozwillo-mongo-2 >> /etc/hosts
Then wait until it's started (docker exec -it ozwillo-mongo-1 tail -f datacore.log)

4. Start virtualbox GUI and setup redirection of port 8080
LATER it would be better to be able to do it using the OCCI configuration of the VM, see #132.

5. In your browser, go to the Datacore Playground at http://localhost:8080/ . The top right dropdown box should list all existing data projects. If you select for instance the "geo_1" project, its "project portal" should be displayed in the central color textarea, and clicking on its first (eponymous) link should display the project's configuration in JSON(-LD) format.

6. In the Linked Data Studio, do the "Publish" action on the "geo_1" project. It should set its "dcmp:frozenModelNames" property to ["*"] (and similarly "Unpublish" should set it to []), which can be seen in the Datacore Playground in said project's configuration.

7. In the Linked Data Studio, do the "Update" action on the "org_1" and "geo_analytics_1" projects. It should create them, meaning they should be listed in the Datacore Playground's project dropdown list after refreshing the page. Their project configuration (if displayed in the Datacore Playground as said) should be the same as the one set in the OCCI configuration. Especially, the org_1's project "dcmpv:name" property should be set to the value of the "occi.ld.project.name" attribute ("org_1"), and its "dcmp:localVisibleProjects" property should be a list of URIs of projects linked by LDProjectLinks in the OCCI configuration.
(For now the Update action is merely doing a "get" then "create" or "merge and update" according to whether the LDProject already exists. LATER the Update action could probably replaced by properly implementing the CRUD > POST action, possibly improved by implementing a "Synchronize" action getting back the current state of the configuration, maybe automatically executed, with "occi.ld.project.version" being -1 for not yet created projects)

8. In the Datacore Playground, select the "geo_analytics_1" project as current project in the top dropdown box. Then write the sample analytics query (*) in the Playground's URL bar and execute it using the "?" debug button. Look up "serverAddress" in the results : its "host" should be set to the host of the Compute is linked to the OCCI LDProject through an LDDatabaseLink if any (the ozwillo-mongo-2 secondary MongoDB host in the demo configuration), and to the primary MongoDB host otherwise (ozwillo-mongo-1 in the demo configuration), as is the case when selecting another project (such as "geo_1").

(*) sample analytics query :
for now : /dc/type/dcmo:model_0?dcmo:isHistorizable=false
LATER energy consumption-specific model and sample data will be provided and will allow for a more meaningful analytics query from a business point of view.


## Do the same with the OW2 Open Stack VM :
In the Docker Studio on said VM (ozwillodatacoredevlocal in the ozwillo-datacore-cluster.docker configuration), first set its "username" and "password" attributes (if you don't have any, you must ask 0W2 at https://jira.ow2.org/browse/SERVICEDESK ).

Then do as detailed for the VirtualBox VM.


## To rewrite the extension from scratch :
If you prefer rewriting the Linked Data extension from scratch rather than using the complete version provided by the demo archive, do as in the EclipseCon slides, with the following hints.

### Extension definition linkeddata.occie :
reuse the provided one, or define it using the Extension designer like this :
extension linkeddata : "http://occiware.org/linkeddata#"
import "http://schemas.ogf.org/occi/core#/"
import "http://schemas.ogf.org/occi/infrastructure#/"
import "http://schemas.ogf.org/occi/platform#/"
kind ldproject extends core.resource {
	title "LDProject"
	attribute occi.ld.project.name : core.String
	attribute occi.ld.project.published : core.Boolean = "false"
	attribute occi.ld.project.robust : core.Boolean = "true"
	action publish ()
	action unpublish ()
	action update ()
}
kind lddatabaselink extends core.link {
	title "LDDatabaseLink"
	attribute occi.ld.dblink.database : core.String = "datacore"
	attribute occi.ld.dblink.port : core.Number = "27017"
}
kind ldprojectlink extends core.link {
}

### Extension connector :
reuse the provided code

### Extension designer :
reuse the provided one, or use the generated one and add
- a copy of the Docker Designer's container Container
- and a Drop Container that targets it, to allow drag'n'dropping Docker containers in the LinkedData Designer 

### Docker configuration occiware-datacore-cluster.docker/occic :
reuse the provided one, or define it using the Docker designer like this :
(only showing the local VirtualBox, but others ex. remote OW2 OpenStack can be created just the same way)
configuration
use "http://occiware.org/occi/docker#/"
resource "6df690d2-3158-40c4-88fb-d1c41584d6e4" : docker.machine_VirtualBox {
	state name = "ozwillodatacoredev"
	state occi.core.id = "6df690d2-3158-40c4-88fb-d1c41584d6e4"
	link "da58c05f-fe96-4fc8-aab9-3e9232aab767" : docker.contains target
	"9dcd39ac-4451-44eb-ac05-227951c23d40" {
		state occi.core.id = "da58c05f-fe96-4fc8-aab9-3e9232aab767"
	}
	link "a92ea98c-ca50-467b-a97c-bba9cc483322" : docker.contains target
	"71405ae7-2402-455c-9c96-0de3e4fc39a6" {
		state occi.core.id = "a92ea98c-ca50-467b-a97c-bba9cc483322"
	}
	link "177a2374-1194-41b0-9f34-0c873b3400bf" : docker.contains target
	"55936644-6215-495d-967f-7d453be484a5" {
		state occi.core.id = "177a2374-1194-41b0-9f34-0c873b3400bf"
	}
	link "fe219da5-79d0-477b-ae3d-f3c976c70d6a" : docker.contains target
	"cc806100-d62a-488f-ac13-eb9b20d2914e" {
		state occi.core.id = "fe219da5-79d0-477b-ae3d-f3c976c70d6a"
	}
}
resource "9dcd39ac-4451-44eb-ac05-227951c23d40" : docker.container {
	state occi.core.id = "9dcd39ac-4451-44eb-ac05-227951c23d40"
	state name = "ozwillo-mongo-1"
	state image = "mdutoo/ozwillo-mongo"
	state occi.compute.hostname = "ozwillo-mongo-1"
}
resource "71405ae7-2402-455c-9c96-0de3e4fc39a6" : docker.container {
	state occi.core.id = "71405ae7-2402-455c-9c96-0de3e4fc39a6"
	state name = "ozwillo-mongo-2"
	state image = "mdutoo/ozwillo-mongo"
	state occi.compute.hostname = "ozwillo-mongo-2"
}
resource "55936644-6215-495d-967f-7d453be484a5" : docker.container {
	state occi.core.id = "55936644-6215-495d-967f-7d453be484a5"
	state name = "ozwillo-mongo-3"
	state image = "mdutoo/ozwillo-mongo"
	state occi.compute.hostname = "ozwillo-mongo-3"
}
resource "cc806100-d62a-488f-ac13-eb9b20d2914e" : docker.container {
	state occi.core.id = "cc806100-d62a-488f-ac13-eb9b20d2914e"
	state name = "ozwillo-datacore-1"
	state image = "mdutoo/ozwillo-datacore:latest"
	state ports = "8080:8080"
	state occi.compute.hostname = "ozwillo-datacore-1"
	link "bf49c770-453c-4a16-a7f0-4f8bae191706" : docker.link target
	"9dcd39ac-4451-44eb-ac05-227951c23d40" {
		state occi.core.id = "bf49c770-453c-4a16-a7f0-4f8bae191706"
	}
	link "fe41e935-801c-41a4-8e2f-5b29daa978ce" : docker.link target
	"55936644-6215-495d-967f-7d453be484a5" {
		state occi.core.id = "fe41e935-801c-41a4-8e2f-5b29daa978ce"
	}
	link "29e9dbc4-44db-48b3-af1d-7d0123bcf3ec" : docker.link target
	"71405ae7-2402-455c-9c96-0de3e4fc39a6" {
		state occi.core.id = "29e9dbc4-44db-48b3-af1d-7d0123bcf3ec"
	}
}

### LinkedData configuration Mytest.linkeddata/occic :
reuse the provided one, or define it using the LinkedData designer like in the EclipseCon slides.

### To rebuild the docker images :

# first build ozwillo datacore :
pushd YOUR_OZWILLO_DATACORE_WORKSPACE
git clone git@github.com:ozwillo/ozwillo-datacore.git
cd ozwillo-datacore
git checkout 2cada878ecaf901fb7750d65b6cda66815467ff2
mvn clean install -DskipTests
popd

# then build & push the docker images :
sudo docker login
# and replace mdutoo by your dockerhub user name in the following command lines

cd docker/ozwillo-datacore
cp -rf YOUR_OZWILLO_DATACORE_WORKSPACE/ozwillo-datacore/ozwillo-datacore-web/target/datacore .
sudo docker build -t mdutoo/ozwillo-datacore:1.0 .
sudo docker push mdutoo/ozwillo-datacore:1.0

cd docker/ozwillo-mongo
sudo docker build -t mdutoo/ozwillo-mongo:1.0 .
sudo docker push mdutoo/ozwillo-mongo:1.0
