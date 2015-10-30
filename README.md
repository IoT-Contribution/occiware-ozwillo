# occiware-ozwillo

To install erocci and to know how this project was developed, please check the ![Tutorial Occiware for ozwillo](./Tutorial_ozwillo.pdf).


### Prototype project directories

* org.ozwillo.data.model.dcproject
 - Contains an OCCI Extension that models resources of Ozwillo platform (occie)
* org.ozwillo.data.ozwillo.samples
 - Contains a sample representation of Ozwillo resources (Project, model, links) in occic format. This representation was developed using the CloudDesigner created by Obeo.
* org.eclipse.acceleo.module.occiware.ozwillo
 - Contains the Ozwillo Curl Generators using the occi configuration "occic", actually allowing for now only to do a POST (genOCCIPost.mtl), and that should later be replaced by the Studio's generic curl generator.
* org.ozwillo.data.designer.docker
 - Contains Ozwillo docker configurations : a drawing of the high-level production architecture, and a configuration of a possible future dockerization of the Ozwillo platform in production.

NB. This projects should be loaded in the Eclipse version built by Obeo (CloudDesigner).


### Docker cofigruation 

The directory that contains the configuration file is named "![docker-erocci-ozwillo](./docker-erocci-ozwillo)".

It contains the files required to configure the docker/erocci :
 - sys.config
 - occie in xml format (which must be setup in sys.config, and has been generated by the Studio)

*Also you can check the ![docker-erocci](https://github.com/erocci/docker-erocci) project to obtain the required files to launch a docker container having the erocci system interface .*
