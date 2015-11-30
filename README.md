# NHS Code4Health Ehrscape Domain


### What has been pre-prepared

1. A Code4Health [Ehrscape](https://ehrscape.code-4-health.org) domain which allows you to control your own patient data and clinical content models.

2. Access to the Marand [EhrExplorer](https://ehrscape.code-4-health.org/explorer) view of your domain. This will allow templates to be viewed and Archetype Query Language (AQL) queries to be constructed.

3. A [Workspace.md](/workspace.md) file containing details of key domain identifiers and assets e.g. dummy patient identifiers, template names, login and password details for your domain.

4. A [local repository](/models) of openEHR clinical content models (archetypes and templates. These archetypes and templates have assembled partly from remote repositories such as the [openEHR Foundation CKM](http://openehr.org/ckm) repository, or [UK Clinical Models](http://clinicmodels.org.uk) repository. Other artefacts, in particular the openEHR templates, have been authored locally and are maintained along with copies of the required archetypes in this Git repository.

	The ‘operational templates’ (effectively the compiled version 	of each template) have been uploaded to your domain along with some dummy data.

The templates are most easily viewed via the Marand [EhrExplorer](https://ehrscape.code-4-health.org/explorer) tool.

5. A [Postman](https://www.getpostman.com/) [Collections](/technical/postman/NHS%20Code4Health%20Ehrscape%20Master.json.postman_collection) file which can be imported to provide easy access to the Code4Health Ehrscape API.

6. A [Postman](https://www.getpostman.com/) [Environment](/technical/postman/C4H.postman_environment) file for your domain, containing a number of preset variables.

7. A [Technical guide](/docs/scenarios/tech_guide.md), describing the key tasks involved in getting up and running with Ehrscape.

8. An [Overview of openEHR and Ehrscape guide](/docs/openehr/openehr_intro.md) for those new to openEHR concepts.

9. A [Code4Health Ehrscape API Reference Guide](https://code-4-health.org/platform/open_interfaces_apis/ehrscape/ehrscape_api_reference) with detailed descriptions of the calls.
