## Code4Health Ehrscape Technical guide

This document describes the series of [Code4Health Ehrscape API](https://ehrscape.code-4-health.org/api-explorer.html) calls required for the LCR Demo Summary app and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open the Ehrscape Session

* Retrieve Patient's ehrId from Ehrscape, based on their `subjectId` (NHSNumber) and subjectNamespace (normally `uk.nhs.nhs_number`)

* Retrieve the compositionID of the patient's most recent `Nursing Vital Signs Observations` Composition.  

* Retrieve the most recent`Nursing Vital Signs Observations` Composition  

* Persist a new `Nursing Vital Signs Observations` Composition  as a POST i.e. create a new instance of the document.  

* Run an AQL query which returns a 'flat list' of key datapoints from the 3 most recent `Nursing Vital Signs Observations` for read-only display.  

* Close the Ehrscape Session


### API parameter conventions

The baseURL for all Ehrscape API calls is https://ehrscape.code-4-health.org

In the descriptions below we use the convention

`{{Password}}`

to indicate that a local variable or parameter should be substituted

The baseURL for all Ehrscape API calls is https://ehrscape.code-4-health.org

i.e. the call below, described as
`POST /rest/v1/session?username={{userName}}&password={{password}}`

where
```
userName=myUserName
password = mypassword
```
should be resolved to
```
https://ehrscape.code-4-health.org/rest/v1/session?username=myuserName&password=mypassword
```

### A. Open the Ehrscape session

The first step in working with Ehrscape is to open a Session and retrieve the `sessionId` token. This allows subsequent API calls to be made without needing to login on each occasion.  
The session should be formally closed when you are finished.

##### Call: Create a new openEHR session:
 ````
 POST /rest/v1/session?username={{userName}}&password={{password}}
 ````
##### Returns:
```
{
"sessionId": "fc234d24-7b59-49f5-a724-8d37072e832b"
}
```

### B. Retrieve Patient’s ehrId from Ehrscape based on their subjectId

We now need to retrieve the patient’s internal `ehrID` associated with their subjectId. The ehrID is a unique string which, for security reasons, cannot be associated with the patient, if for instance their openEHR records were leaked.

##### Call: Returns the EHR for the specified subject ID and namespace.
```
GET /rest/v1/ehr/?subjectId={{subjectId}}&subjectNamespace={{subjectNamespace}}
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
```
##### Return:
```
{
  “ehrId”: “0da489ee-c0ae-4653-9074-57b7f63c9f16”
}
```

### C. Retrieve the compositionId of Patient’s most recent `Nursing Vital Signs Observations` Composition

Now that we have the patient’s `ehrId` we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing `Nursing Vital Signs Observations` Composition records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the templates composition archetype (case-sensitive). In a real-world example we would query on other factors to ensure we had the ‘correct’ list.

##### AQL statement

```
select
    a/uid/value as compositionId,
    a/context/start_time/value as start_time
from EHR e[ehr_id/value='{{ehrId}}']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1]
where a/name/value= 'Nursing Vital Signs Observations'
order by a/context/start_time/value desc
offset 0 limit 1
```  

The query API call returns a `resultSet` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `compositionId` element in the response is the unique identifier for the composition and `start_time` is the time that the document was authored.

##### Call: Run AQL query and return a Resultset
```
GET /rest/v1/query?aql=select a/uid/value as compositionId, a/context/start_time/value as start_time from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] where a/name/value= 'Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 1
Authorization: Basic eWRoOnBIaVNtb25h
```
##### Return:
```
{
    "meta": {
        "href": "http://ehrscape.code-4-health.org/rest/v1/query/?aql=select%20%20%20%20%20a/uid/value%20as%20compositionId,%20%20%20%20%20a/context/start_time/value%20as%20start_time%20from%20EHR%20e%5Behr_id/value%3D'dd0901e5-a134-465d-9e6d-23a125f5d1d4'%5D%20contains%20COMPOSITION%20a%5BopenEHR-EHR-COMPOSITION.encounter.v1%5D%20where%20a/name/value%3D%20'Nursing%20Vital%20Signs%20Observations'%20order%20by%20a/context/start_time/value%20desc%20offset%200%20limit%201"
    },
    "aql": "select     a/uid/value as compositionId,     a/context/start_time/value as start_time from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] where a/name/value= 'Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 1",
    "executedAql": "select     a/uid/value as compositionId,     a/context/start_time/value as start_time from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] where a/name/value= 'Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 1",
    "resultSet": [
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "compositionId": "3a0f1a4a-c89f-48cc-a4bf-1c1c14b48587::ydh.code4health.com::1"
        }
    ]
}
```

### C. Retrieve the Patient’s most recent `Nursing Vital Signs Observations` Composition

We will use the results of the previous query to retrieve one of the compositions via its `compositionId`.

##### Call: Returns the specified Composition in FLAT JSON format.

```
GET /rest/v1/composition/3a0f1a4a-c89f-48cc-a4bf-1c1c14b48587::ydh.code4health.com::1?format=FLAT HTTP/1.1
Headers:
 Content-Type: application/json
 Ehr-Session: {{sessionId}} //The value of the sessionId
```
##### Return:
```
{
    "meta": {
        "href": "http://ehrscape.code-4-health.org/rest/v1/composition/3a0f1a4a-c89f-48cc-a4bf-1c1c14b48587::ydh.code4health.com::1"
    },
    "format": "FLAT",
    "templateId": "Vital Signs Encounter (Composition)",
    "composition": {
        "nursing_vital_signs_observations/_uid": "3a0f1a4a-c89f-48cc-a4bf-1c1c14b48587::ydh.code4health.com::1",
        "nursing_vital_signs_observations/context/start_time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/context/setting|code": "238",
        "nursing_vital_signs_observations/context/setting|value": "other care",
        "nursing_vital_signs_observations/context/setting|terminology": "openehr",
        "nursing_vital_signs_observations/vital_signs:0/respirations:0/any_event:0/rate|magnitude": 17,
        "nursing_vital_signs_observations/vital_signs:0/respirations:0/any_event:0/rate|unit": "/min",
        "nursing_vital_signs_observations/vital_signs:0/respirations:0/any_event:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/vital_signs:0/pulse:0/any_event:0/heart_rate|magnitude": 89,
        "nursing_vital_signs_observations/vital_signs:0/pulse:0/any_event:0/heart_rate|unit": "/min",
        "nursing_vital_signs_observations/vital_signs:0/pulse:0/any_event:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/vital_signs:0/body_temperature:0/any_event:0/temperature|magnitude": 36.6,
        "nursing_vital_signs_observations/vital_signs:0/body_temperature:0/any_event:0/temperature|unit": "°C",
        "nursing_vital_signs_observations/vital_signs:0/body_temperature:0/any_event:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/systolic|magnitude": 104,
        "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/systolic|unit": "mm[Hg]",
        "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/diastolic|magnitude": 60,
        "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/diastolic|unit": "mm[Hg]",
        "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/spo2|numerator": 98,
        "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/spo2|denominator": 100,
        "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/spo2": 0.98,
        "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/vital_signs:0/national_early_warning_score_rcp_uk:0/total_score": 0,
        "nursing_vital_signs_observations/vital_signs:0/national_early_warning_score_rcp_uk:0/time": "2015-10-10T02:19:00.000Z",
        "nursing_vital_signs_observations/composer|name": "ydh"
    },
    "deleted": false,
    "lastVersion": true
}
```

### D. Persist an new instance of the `Nursing Vital Signs Observations` Composition as a POST

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  ‘target composition’ data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

In this scenario every Vital signs Bursing observation merits a separate instance of the document and a POST is used, with PUT updates only being used to correct errors or to perform ‘sign-off’.

In other scenarios we might want to, for example, **maintain only a single instance of an Allergies List** and so will use a PUT call to update the previous version. The previous version remains available for audit purposes but would not normally be found by routine querying.
**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

##### Call: Creates a new openEHR composition and returns the new CompositionId
```
POST /rest/v1/composition?ehrId=7dfbbeab-d49e-42bf-9f6e-a23d5b55e812&templateId=Vital Signs Encounter (Composition)&format=FLAT

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json

Body:
{
 “ctx/language”: “en”,
 “ctx/territory”: “GB”,
 “ctx/composer_name”: “Hazel Smith”,
 “ctx/time”: “"2015-12-10T02:19:00.000Z”,
 "ctx/health_care_facility|id": "999999-345",
 "ctx/health_care_facility|name": "Northumbria Community NHS",
 "ctx/id_namespace": "NHS-UK",
 "ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
    "nursing_vital_signs_observations/vital_signs:0/respirations:0/any_event:0/rate|magnitude": 22,
    "nursing_vital_signs_observations/vital_signs:0/respirations:0/any_event:0/rate|unit": "/min",
    "nursing_vital_signs_observations/vital_signs:0/pulse:0/any_event:0/heart_rate|magnitude": 101,
    "nursing_vital_signs_observations/vital_signs:0/pulse:0/any_event:0/heart_rate|unit": "/min",
    "nursing_vital_signs_observations/vital_signs:0/body_temperature:0/any_event:0/temperature|magnitude": 36.6,
    "nursing_vital_signs_observations/vital_signs:0/body_temperature:0/any_event:0/temperature|unit": "°C",
    "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/systolic|magnitude": 100,
    "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/systolic|unit": "mm[Hg]",
    "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/diastolic|magnitude": 60,
    "nursing_vital_signs_observations/vital_signs:0/blood_pressure:0/any_event:0/diastolic|unit": "mm[Hg]",
    "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/spo2|numerator": 94,
    "nursing_vital_signs_observations/vital_signs:0/indirect_oximetry:0/any_event:0/spo2|denominator": 100,
    "nursing_vital_signs_observations/vital_signs:0/national_early_warning_score_rcp_uk:0/total_score": 3,
  }
```
##### Return:  

```
  {
      “meta”: {
          “href”: “https://ehrscape.code-4-health.org/rest/v1/composition/c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”
      },
      “action”: “CREATE”,
      “compositionUid”: “c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”
  }
```

### D. Run an AQL query with key values

This AQL query returns a flattened Resultset with key values from recent compositions.  

##### Call: Run AQL query and return a Resultset

This AQL call returns a set of key data points from recent `Nursing Vital Signs Observations` Compositions. The `resultSet` is a simple nested name/value pair structure whose exact format is determined by the query itself. Either single data points or objects can be  returned. In this case we will select single, datapoints.

```
GET /rest/v1/query?aql=select     a/uid/value as compositionId,     a/context/start_time/value as start_time,     b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/magnitude as Rate_magnitude,     b_b/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Heart_Rate_magnitude,     b_c/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Temperature_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as Systolic_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as Diastolic_magnitude,     b_g/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator as spO2_numerator,     b_h/data[at0001]/events[at0002]/data[at0003]/items[at0028]/value/magnitude as Total_Score_magnitude from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] contains (     OBSERVATION b_a[openEHR-EHR-OBSERVATION.respiration.v1] or     OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulse.v1] or     OBSERVATION b_c[openEHR-EHR-OBSERVATION.body_temperature.v1] or     OBSERVATION b_f[openEHR-EHR-OBSERVATION.blood_pressure.v1] or     OBSERVATION b_g[openEHR-EHR-OBSERVATION.indirect_oximetry.v1] or     OBSERVATION b_h[openEHR-EHR-OBSERVATION.news_rcp_uk.v1]) where a/name/value='Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 5 HTTP/1.1

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
```

##### Return:
```
{
    "meta": {
        "href": "http://ehrscape.code-4-health.org/rest/v1/query/?aql=select%20%20%20%20%20a/uid/value%20as%20compositionId,%20%20%20%20%20a/context/start_time/value%20as%20start_time,%20%20%20%20%20b_a/data%5Bat0001%5D/events%5Bat0002%5D/data%5Bat0003%5D/items%5Bat0004%5D/value/magnitude%20as%20Rate_magnitude,%20%20%20%20%20b_b/data%5Bat0002%5D/events%5Bat0003%5D/data%5Bat0001%5D/items%5Bat0004%5D/value/magnitude%20as%20Heart_Rate_magnitude,%20%20%20%20%20b_c/data%5Bat0002%5D/events%5Bat0003%5D/data%5Bat0001%5D/items%5Bat0004%5D/value/magnitude%20as%20Temperature_magnitude,%20%20%20%20%20b_f/data%5Bat0001%5D/events%5Bat0006%5D/data%5Bat0003%5D/items%5Bat0004%5D/value/magnitude%20as%20Systolic_magnitude,%20%20%20%20%20b_f/data%5Bat0001%5D/events%5Bat0006%5D/data%5Bat0003%5D/items%5Bat0005%5D/value/magnitude%20as%20Diastolic_magnitude,%20%20%20%20%20b_g/data%5Bat0001%5D/events%5Bat0002%5D/data%5Bat0003%5D/items%5Bat0006%5D/value/numerator%20as%20spO2_numerator,%20%20%20%20%20b_h/data%5Bat0001%5D/events%5Bat0002%5D/data%5Bat0003%5D/items%5Bat0028%5D/value/magnitude%20as%20Total_Score_magnitude%20from%20EHR%20e%5Behr_id/value%3D'dd0901e5-a134-465d-9e6d-23a125f5d1d4'%5D%20contains%20COMPOSITION%20a%5BopenEHR-EHR-COMPOSITION.encounter.v1%5D%20contains%20(%20%20%20%20%20OBSERVATION%20b_a%5BopenEHR-EHR-OBSERVATION.respiration.v1%5D%20or%20%20%20%20%20OBSERVATION%20b_b%5BopenEHR-EHR-OBSERVATION.pulse.v1%5D%20or%20%20%20%20%20OBSERVATION%20b_c%5BopenEHR-EHR-OBSERVATION.body_temperature.v1%5D%20or%20%20%20%20%20OBSERVATION%20b_f%5BopenEHR-EHR-OBSERVATION.blood_pressure.v1%5D%20or%20%20%20%20%20OBSERVATION%20b_g%5BopenEHR-EHR-OBSERVATION.indirect_oximetry.v1%5D%20or%20%20%20%20%20OBSERVATION%20b_h%5BopenEHR-EHR-OBSERVATION.news_rcp_uk.v1%5D)%20where%20a/name/value%3D'Nursing%20Vital%20Signs%20Observations'%20order%20by%20a/context/start_time/value%20desc%20offset%200%20limit%205"
    },
    "aql": "select     a/uid/value as compositionId,     a/context/start_time/value as start_time,     b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/magnitude as Rate_magnitude,     b_b/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Heart_Rate_magnitude,     b_c/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Temperature_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as Systolic_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as Diastolic_magnitude,     b_g/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator as spO2_numerator,     b_h/data[at0001]/events[at0002]/data[at0003]/items[at0028]/value/magnitude as Total_Score_magnitude from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] contains (     OBSERVATION b_a[openEHR-EHR-OBSERVATION.respiration.v1] or     OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulse.v1] or     OBSERVATION b_c[openEHR-EHR-OBSERVATION.body_temperature.v1] or     OBSERVATION b_f[openEHR-EHR-OBSERVATION.blood_pressure.v1] or     OBSERVATION b_g[openEHR-EHR-OBSERVATION.indirect_oximetry.v1] or     OBSERVATION b_h[openEHR-EHR-OBSERVATION.news_rcp_uk.v1]) where a/name/value='Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 5",
    "executedAql": "select     a/uid/value as compositionId,     a/context/start_time/value as start_time,     b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/magnitude as Rate_magnitude,     b_b/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Heart_Rate_magnitude,     b_c/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Temperature_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as Systolic_magnitude,     b_f/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as Diastolic_magnitude,     b_g/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator as spO2_numerator,     b_h/data[at0001]/events[at0002]/data[at0003]/items[at0028]/value/magnitude as Total_Score_magnitude from EHR e[ehr_id/value='dd0901e5-a134-465d-9e6d-23a125f5d1d4'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] contains (     OBSERVATION b_a[openEHR-EHR-OBSERVATION.respiration.v1] or     OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulse.v1] or     OBSERVATION b_c[openEHR-EHR-OBSERVATION.body_temperature.v1] or     OBSERVATION b_f[openEHR-EHR-OBSERVATION.blood_pressure.v1] or     OBSERVATION b_g[openEHR-EHR-OBSERVATION.indirect_oximetry.v1] or     OBSERVATION b_h[openEHR-EHR-OBSERVATION.news_rcp_uk.v1]) where a/name/value='Nursing Vital Signs Observations' order by a/context/start_time/value desc offset 0 limit 5",
    "resultSet": [
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "spO2_numerator": 98,
            "Total_Score_magnitude": 0,
            "Rate_magnitude": 17,
            "compositionId": "3a0f1a4a-c89f-48cc-a4bf-1c1c14b48587::ydh.code4health.com::1",
            "Diastolic_magnitude": 60,
            "Heart_Rate_magnitude": 89,
            "Systolic_magnitude": 104,
            "Temperature_magnitude": 36.6
        },
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "spO2_numerator": 98,
            "Total_Score_magnitude": 0,
            "Rate_magnitude": 17,
            "compositionId": "4b31520c-0f31-411b-a05d-132ce60be71e::ydh.code4health.com::1",
            "Diastolic_magnitude": 60,
            "Heart_Rate_magnitude": 89,
            "Systolic_magnitude": 104,
            "Temperature_magnitude": 36.6
        },
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "spO2_numerator": 98,
            "Total_Score_magnitude": 0,
            "Rate_magnitude": 17,
            "compositionId": "05611b6b-571c-4232-80d1-be297669cc61::ydh.code4health.com::1",
            "Diastolic_magnitude": 60,
            "Heart_Rate_magnitude": 89,
            "Systolic_magnitude": 104,
            "Temperature_magnitude": 36.6
        },
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "spO2_numerator": 98,
            "Total_Score_magnitude": 0,
            "Rate_magnitude": 17,
            "compositionId": "81ac0db3-b2a3-4e25-9d54-a1ab22b492ef::ydh.code4health.com::1",
            "Diastolic_magnitude": 60,
            "Heart_Rate_magnitude": 89,
            "Systolic_magnitude": 104,
            "Temperature_magnitude": 36.6
        },
        {
            "start_time": "2015-10-10T02:19:00.000Z",
            "spO2_numerator": 98,
            "Total_Score_magnitude": 0,
            "Rate_magnitude": 17,
            "compositionId": "a09d5190-0974-4d9c-8c9d-3f68cc15beb1::ydh.code4health.com::1",
            "Diastolic_magnitude": 60,
            "Heart_Rate_magnitude": 89,
            "Systolic_magnitude": 104,
            "Temperature_magnitude": 36.6
        }
    ]
}
```

### E. Close the Ehrscape session

The last step in working with Ehrscape is to close the session.

##### Call: Close the openEHR session:
 ````
 DELETE /rest/v1/session?sessionId={{sessionId}}
 ````
##### Returns:
````
{
  "sessionId": "2dcd6528-0471-4950-82fa-a018272f1339"
}
````
