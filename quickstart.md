Adobe Task Queue Manager getting started guide
==============================================

Introduction
------------

This guide describes how to setup a system to allow access to the cloud API of
the Adobe Task Queue Manager and perform a set of basic requests.

Features
--------

Task Queue Manager allows customers to send jobs from one machine to another
while receiving real time updates about progress. These jobs can either be
encode jobs (similar to Adobe Media Encoder Remote) or script jobs with
arbitrary extension code. The transfer and delivery from the client to the
server machine is handled by queues in the cloud. A server/worker can be
deployed to any computer running PremierePro or Adobe Media Encoder. The content
of e.g. encode jobs is composed of the project file (or team project snapshot)
an optional export preset file and meta data.

Installing a worker
-------------------

-   Install Java jdk 11

-   Install Premiere Pro 14.0 and Adobe Media Encoder 14.0 (2020)

-   The worker can be installed by extracting the zip file to any folder on a
    machine. It can also be setup to run as a service or started as a process.

-   Retrieve the account credentials from console.adobe.io and configure the
    cc-worker.xml file

-   Make sure there is a queue already created to be set on the worker (created
    via API or from Premiere Pro DOM calls)

Account setup
-------------

Login using an administrator account of your domain on
https://console.adobe.io/. And choose to create a new integration.

On the following screen choose "Access an API"

Choose "Task Queue Manager" from the list.

In the following screen you can name the integration and provide a security
certificate used for authentication - see the "Creating the worker keystore file
and certificate" chapter in the developer guide below or

https://www.adobe.io/authentication/auth-methods.html\#!AdobeDocs/adobeio-auth/master/AuthenticationOverview/ServiceAccountIntegration.md\#step-2-
create-a-public-key-certificate. Drag the "certificate_pub.crt" into the drop
box and continue.

The integration is now created and you can use the following page to retrieve
all configuration values needed to setup a worker. (API_ADOBE_USER_ID in the
config file refers to the Technical account ID on this page)

### Running as a process (for debugging only)

Update the run-as-process.bat file found in the worker directory with the
information found on the adobe.io developer console. Then start the bat file.
Make sure java RE \>= 11 is installed and available on the PATH.

### Running as a service

The configuration file cc-worker.xml can be found in the service folder. The XML
lists all required values which were generated in the previous step for creating
a JWT token on Adobe.io. Make sure to use a local user which has sufficient
rights to start both Premiere Pro and Adobe Media Encoder. The service should
not be run on the default service account.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
<service>
<id>dva-worker</id>
<name>Adobe Task Queue Manager</name>
<description>This service runs the Adobe Task Queue Manager</description>
<env name="WORKER_HOME" value="%BASE%"/>
<env name="API_ADOBE_USER_ID" value="user-id-from-adobe-io-console"/>
<env name="API_KEY" value="api-key-from-adobe-io-console"/>
<env name="API_ADOBE_ORG_ID" value="org-id-from-adobe-io-console"/>
<env name="API_CLIENTSECRET" value="clientsecret-from-adobe-io-console"/>
<env name="QUEUE_ID" value="the-queue-id-to-process" />
<env name="API_KEYSTORE_FILE" value="%BASE%\keystore.ks" />
<env name="API_KEYSTORE_PASSWORD" value="changeme" />
<!-- <env name="API_ENDPOINT" value="cloud-dispatcher-beta.adobe.io" /> -->
<executable>java</executable>
<arguments>-Djava.io.tmpdir=%TEMP% -Xmx1024m -jar "%BASE%\worker.jar"</arguments>
<onfailure action="restart" delay="10 sec"/>
<log mode="roll-by-time">
<pattern>yyyyMMdd</pattern>
</log>
<logpath>%BASE%\logs</logpath>
<serviceaccount>
<domain>YOURDOMAIN</domain>
<user>useraccount</user>
<password>password or %PASSWORD_FROM_ENVIRONMENT_VARIABLE%</password>
<allowservicelogon>true</allowservicelogon>
</serviceaccount>
</service>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The process can be setup to run as a service as soon as the configuration file
and keystore file are correctly setup:

This requires Admin rights on the local machine

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cc-worker.exe install
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cc-worker.exe start
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The process will populate log files to the /logs directory of the same folder.
Make sure the process is running as a user which is allowed to write to that
folder, and has rights to start Adobe Media Encoder. The correct user can be
verified when opening the service list, opening the "Adobe Task Queue Manager"
service and inspecting the "log on" tag. It should show the service using a
specific user account instead of "Local system account".

Please note cc-worker.exe install has to be repeated every time the user
password is changed in the configuration.

### Prerequisite:

Task Queue Manager features are supported only on 'New World' scripting engine
which is not the default on Premiere Pro/ AME 14.0.0 builds. Therefore in order
to use Task Queue Manager, New World needs to be enabled on all clients and
worker machine using one of the following methods.

1.  Quit Premiere Pro/ Media Encoder. Copy attached debug files for AME and
    Premiere Pro and replace them at following location.

| Stable Builds | Media Encoder                                            | Premiere Pro                                                |
|---------------|----------------------------------------------------------|-------------------------------------------------------------|
| Windows       | C:\<UserName\>Media Encoder4.0                           | C:Pro4.0                                                    |
| Mac           | /Users//Documents/Adobe/Adobe Media Encoder/14.0/        | /Users//Library/Preferences/Adobe/Premiere Pro/14.0/        |

| Beta Builds   | Media Encoder                                            | Premiere Pro                                                |
|---------------|----------------------------------------------------------|-------------------------------------------------------------|
| Windows       | C:\<UserName\>Media Encoder (Beta)4.0                    | C:\<UserName\>Pro (Beta)4.0                                 |
| Mac           | /Users//Documents/Adobe/Adobe Media Encoder (Beta)/14.0/ | /Users//Library/Preferences/Adobe/Premiere Pro (Beta)/14.0/ |

The debug files changes following flag with respect to each product:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Premiere Pro: ScriptLayerPPro.EnableNewWorld to true

Media Encoder: AME.ScriptLayer.EnableNewWorld to true
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Attached debug database file will only enable 'New World' scripting engine and
rest of debug flags default to original values. If other debug flags are also
being used, they need to be taken care of also.

2.  Launch Premiere Pro. Open any project. Press
    CTRL+F12(Windows)/Command+Fn+F12(Mac) that will open Console Window. Click
    on hamburger menu next to Console and select Debug Database View. Search for
    debug flag ScriptLayerPPro.EnableNewWorld and change it to true by selecting
    the checkbox. New settings will take place on re launch. Launch Adobe Media
    Encoder, Press CTRL+F12 that will open Console Window. Click on hamburger
    menu next to Console and select Debug Database View. Search for debug flag
    AME.ScriptLayer.EnableNewWorld and change it to true by selecting the
    checkbox. New settings will take place on re launch.

Developer guide
---------------

### Generating an API token

For testing purpose you can use the Premiere Pro scripting engine to request a
token for the currently logged in user. In production this workflow will be
replaced
with:https://www.adobe.io/apis/cloudplatform/console/authentication/createjwt.html.
The integration on Adobe.Io can be created via:https://console
.adobe.io/integrations.

In the ExtendScript Toolkit connect to Premiere Pro and invoke the following
command:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
taskQueueManagerObj = qe.tqm.logIntoTaskQueueManager();
var authTokenObj = qe.tqm.getAuthToken();
var bearerToken = authTokenObj.token;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Accessing the API

The API can be accessed via the URL: https://cloud-dispatcher-beta.adobe.io

The API uses the JSON+HAL format
(https://en.wikipedia.org/wiki/Hypertext_Application_Language).

It requires the "Authorization" header to be set to "Bearer
yourBearerTokenFromExtendScriptHere" and the "X-Api-Key" header set to
"task-queue- manager-beta-1".

### Using the Task Queue Manager API from within Premiere Pro (DOM API)

This example script will lookup a queue from the list of existing queues
(without creating a new one). A typical workflow would involve one central queue
being created by an administrator (assigned to the organisation) which all users
can share for submitting work to the central pool of workers.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
qe.tqm.logIntoTaskQueueManager();
var queueId = "same-queue-id-as-used-in-the-worker";
var queueList = qe.tqm.getQueueListObject();
queueList.loadSync();
var queue = null;

var queues = queueList.getQueues();
for (var i=0; i < queues.length; i++) {
if (queues[i].identifier == queueId) {
queue = queues[i];
break;
}
}

var job = queue.createRemoteAMERenderJob(
"example encode job 1",
"c:\\full\\path\\to\\team\\projects\\snapshot\\tp-snapshot.tp2snap",
"c:\\full\\path\\to\\team\\projects\\snapshot\\ExportPresetH264MatchSequence.epr",
"c:\\worker\\harddrive\\or\\network\\share\\output.mp4"
);
var result = job.loadSync();
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### API

All API calls are scoped to the qe dom.

#### Root Level Methods

##### qe.tqm.logIntoTaskQueueManager(callback-method-name) - returns Boolean (success)

used to create an authentication token from the currently logged in user to
communicate the the TQM API

##### qe.tqm.loginByRegion(regionName) -returns Boolean (success)

used to create an authentication token from the currently logged in user on the
region provided as string parameter to communicate the the TQM API

##### qe.tqm.getActiveRegion(); - returns String

returns the active region user is currently logged in.

##### qe.tqm.getQueueListObject() - returns QueueListObject

creates a request object to read the list of queues visible to the logged in
user

##### qe.tqm.createQueue(queueName, ownerId) returns QueueObject

create a request with the given parameters for creating a new queue, owned by
the user (if a blank ownerId is given) or the organisation of the ownerId

##### qe.tqm.getAuthToken() - returns AuthenticationObject

returns the Authentication object which can be then used to get current
authentication token usingAuthenticationObject.token(only available after
calling the logIntoTaskQueueManager method)

##### qe.tqm.setDiscoveryURL(String) - return Boolean (success)

sets the base discovery URL used to communicating with the api (can e.g. be used
to switch to a different region or localhost for debugging)

#### Queue List Level Methods

##### queueList.getQueues() returns an array of Queue objects

after the request was loaded (async/sync) the list of queues is available in
this array

##### queueList.getQueueByID(queueId) returns a Queue Object

directly read a queue object by queueId

##### queueList.loadAsync(callback-method-name) returns Boolean (success)

process the request async (non blocking)

##### queueList.loadSync() returns Boolean (success)

process the request sync (blocking)

#### Queue Level Methods

##### queue.identifier - returns a String identifier property

the ID property of the queue

##### queue.name - returns a String name of the queue

the name property of the queue

##### queue.createRemoteAMERenderProjectJob(jobName, localProjectFilePath, itemGUID, localExportPresetPath,

##### remoteOutputFilePath) - returns a Job request object

creates a project file based job and attaches both the export preset and the
project to the request object.

##### queue.prepareRemoteAMERenderProjectJob(jobName, localProjectFilePath, itemGUID, localExportPresetPath,

##### remoteOutputFilePath) - returns a Job request object

createsproject file based job that will not be automatically be started and
attaches both the export preset and the project to the request object.

##### queue.createRemoteAMERenderJob(jobName, localTeamProjectSnapshotPath, localExportPresetPath, remoteOutputFilePath) -

##### returns a Job request object

creates a team projects snapshot based job and attaches both the snapshot and
export preset to the request object.

##### queue.prepareRemoteAMERenderJob(jobName, localTeamProjectSnapshotPath, localExportPresetPath,

##### remoteOutputFilePath) - returns a Job request object

creates a team projects snapshot based job that will not be automatically be
started and attaches both the snapshot and export preset to the request object.

##### queue.loadSync() - returns queue id(success)

process the request sync (blocking)

##### queue.loadAsync() - returns a Boolean (success)

process the request async (non-blocking)

#### Job Level Methods

##### job.identifier- returns a String identifier property

the ID property of the job

##### job.name- returns a String name property

the Name property of the job

##### job.created- returns a String created property

the creation timestamp of the job in the 'YYYY-MM-DDTHH:MM:SS.XXXZ'format.

##### job.estimatedCompletion- returns a String created property

the estimated completion time inthe 'YYYY-MM-DDTHH:MM:SS.XXXZ'format.

##### job.lastModified- returns a String lastModified property

the last modification timestamp of the job in
the'YYYY-MM-DDTHH:MM:SS.XXXZ'format.

##### job.estimatedCompletion- returns a String estimatedCompletion property

the estimated-completion timestamp of the job in the'YYYY-MM-DDTHH:MM:SS.XXXZ'
format.

##### job.state- returns a String state property

the current state of the job (i.e WAITING, PROGRESSING, COMPLETED, CANCELED)

##### job.type- returns a String type property

the type of the job. (i.e AME.2020.encode, AE.2020.render, PPRO.2020.ingest,
etc)

##### job.lockId- returns a String lockId property

the lockId of the job.

##### job.progress- returns a Float progress property

the current progress value (0-100) of the job.

##### job.progressState- returns a String progressState property

the current state of progress of the job. (i.e. Error, Done, None)

##### job.isCompleted - returns Boolean (success)

status completion of job.(i.e. true, false)

##### job.isError - returns Boolean (success)

status completion if job is in error state or not.(i.e. true, false)

##### job.addInput(name, pathToLocalFile) - returns Boolean (success)

adds an input file the the job - (e.g. if the job is created via
prepareRemoteAMERenderJob - the job will not be started and may receive
additional input files, and can then be started via the job.start() call)

##### job.start() - returns a Boolean (success)

start the job

##### job.cancel() - returns a Boolean (success)

cancel the job

##### job.loadSync()- returns a job id(success)

process the request sync (blocking)

##### job.loadSync(job.identifier) - take job id as string input andreturns a job id(success) and refreshes the property of job passed as input

process the request sync (blocking)

##### job.loadAsync()- returns a Boolean (success)

process the request async (non-blocking)

#### Additional Snapshot Methods

##### app.project.rootItem.children[childNumber].saveProjectSnapshot(snapShotPath)

take childNumber as integer, snapShotPath as string and returns a Boolean
(success)

##### app.anywhere.openTeamProjectSnapshot(snapShotPath)

take snapShotPath as string and returns a Boolean (success)

##### project.applyProjectSnapshot(snapShotPath, keepSessionProperties,deleteAssetsNotInSnapshot)

take snapShotPath as string, keepSessionProperties as boolean,
deleteAssetsNotInSnapshot as boolean returns a Boolean (success)

The snapshot contents will be merged into the currently-open Team Project. If
keepSessionProperties is set to true, the existing properties will not be
replaced; otherwise, the project name and description will be replaced by those
of the snapshot. If deleteAssetsNotInSnapshot is set to true, any assets not in
the snapshot will be deleted; otherwise, the snapshot will be merged into the
existing project (snapshot assets will overwrite any existing identical assets).

Adobe Task Queue Manager API documentation
------------------------------------------

### Getting started with the Adobe Task Queue Manager API. The API uses the HAL JSON format: http://stateless.co/hal_specification.html. The API can be navigated by following the links starting from the discovery endpoint.

### Authentication

The API supports authentication through a bearer token header and an api key.
This can be generated via a JWT token workflow using the adobe.io integration:
https://www.adobe.io/apis/cloudplatform/console/authentication/gettingstarted.html
. Setting both the 'Authorization' and the 'X-Api-Key' header is required for
every request.

### Creating the worker keystore file and certificate

```
openssl req -nodes -text -x509 -newkey rsa:2048 -keyout secret.pem -out certificate.pem -days 356

openssl pkcs8 -topk8 -inform PEM -outform DER -in secret.pem  -nocrypt > secret.key

openssl pkcs12 -export -in certificate.pem -inkey secret.pem   -out my.p12

keytool -importkeystore \
        -deststorepass 123456 -destkeypass 123456 -destkeystore keystore.ks \
        -srckeystore my.p12 -srcstoretype PKCS12 -srcstorepass 123456

keytool -changealias -keystore keystore.ks -alias 1 -destalias AdobePrivateKey
```

Resources
---------

### Discovery Links

-   GET /

Returns the list of all links available on the root API.

#### Path parameters

No parameters.

#### Query parameters

No parameters.

#### Request fields

No request body.

#### Response fields

| Path         | Type   | Optional | Description |
|--------------|--------|----------|-------------|
| activeRegion | String | true     |             |

#### Example request

\$ curl 'https://cloud-dispatcher-beta.adobe.io/' -i -X GET  
-H 'X-Api-Key: your-api-key'  
-H 'Authorization: Bearer the-access-token'

#### Example response

HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-8

Content-Length: 797

```{ "activeRegion" : "region-unknown", "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io" }, "jobs" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/{?page,size}", "templated" : true }, "queues" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/{?page,size}", "templated" : true }, "workers" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/workers/{?page,size}", "templated" : true }, "region-ue1" : { "href" : "https://cloud-dispatcher-beta-ue1.adobe.io" }, "region-ew1" : { "href" : "https://cloud-dispatcher-beta-ew1.adobe.io" }, "region-an1" : { "href" : "https://cloud-dispatcher-beta-an1.adobe.io" } } }
```

### List My jobs

-   GET /api/v1/jobs/

#### Path parameters

No parameters.

#### Query parameters

| Path | Type    | Optional | Description          |
|------|---------|----------|----------------------|
| page | Integer | true     | Default value: '0'.  |
| size | Integer | true     | Default value: '50'. |

#### Request fields

No request body.

#### Response fields

| Path                          | Type          | Optional | Description |
|-------------------------------|---------------|----------|-------------|
| content                       | Array[Object] | true     |             |
| content[].lastModified        | String        | false    |             |
| content[].progressState       | String        | true     |             |
| content[].estimatedCompletion | String        | true     |             |
| content[].created             | String        | false    |             |
| content[].lockId              | String        | true     |             |
| content[].progress            | Decimal       | true     |             |
| content[].name                | String        | true     |             |
| content[].state               | String        | false    |             |
| content[].type                | String        | false    |             |
| content[].id                  | String        | false    |             |
| page                          | Object        | true     |             |
| page.size                     | Integer       | true     |             |
| page.totalElements            | Integer       | true     |             |
| page.totalPages               | Integer       | true     |             |
| page.number                   | Integer       | true     |             |

#### Example request

\$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/' -i -X GET \\

\-H 'X-Api-Key: your-api-key' \\

\-H 'Authorization: Bearer the-access-token'

#### Example response

HTTP/1.1 200 OK

Content-Length: 2078

Content-Type: application/hal+json;charset=UTF-8

```
{ "_embedded" : { "jobResourceList" : [ { "lastModified" : "2019-09-18T08:37:58.577138Z", "progressState" : null, "estimatedCompletion" : null, "created" : "2019-09-18T08:37:58.577133Z", "lockId" : "workerId", "progress" : 0.0, "name" : "Encode Project to MP4", "state" : "WAITING", "type" : "AME.2020.encode", "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573" }, "inputs" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/inputs" }, "inputs-presign" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/inputs/presign" }, "results" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/results" }, "results-presign" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/results/presign" }, "waitForCancel" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/waitFor/canceled" }, "job-updates" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/2e46f6a6-a933-4192-b609-d3bb40242573/updatedSince/2019-09-18T08:37:58.577138Z" } }, "id" : "2e46f6a6-a933-4192-b609-d3bb40242573" } ] }, "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/?page=0&size=50&sort=lastModified,desc" }, "byId" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/{id}", "templated" : true }, "updates" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/since/2019-09-18T08:37:58.577138Z/" } }, "page" : { "size" : 50, "totalElements" : 1, "totalPages" : 1, "number" : 0 } }
```


### Create Input

-   POST /api/v1/jobs/{id}/inputs/presign

#### Path parameters

| Parameter | Type   | Optional | Description |
|-----------|--------|----------|-------------|
| id        | Object | false    |		      |


#### Query parameters

No parameters

#### Request fields

| Path                 | Type            | Optional | Description |
|----------------------|-----------------|----------|-------------|
| payloads             | Array\[Object\] | true     |		   | 	
| payloads\[\]\.index  | Integer         | true     |		   |
| payloads\[\]\.length | Integer         | true     |		   |
| payloads\[\]\.name   | String          | true     |		   |


#### Response fields

| Path                                    | Type            | Optional | Description |
|-----------------------------------------|-----------------|----------|-------------|
| signedPayloadPairs                      | Array\[Object\] | true     |             |
| signedPayloadPairs\[\]\.index           | Integer         | true     |             |
| signedPayloadPairs\[\]\.requestIndex    | Integer         | true     |             |
| signedPayloadPairs\[\]\.signedUploadURL | String          | true     |             |


#### Example request

curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/dd7d9b8e-f315-45dc-af9d-641f07e4b6e1/inputs/presign' -i -X POST \
    -H 'Content-Type: application/json' \
    -d '{"payloads":[{"index":1,"length":123,"name":"input.prproj"}]}'

#### Example response

HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 124

```
{ "signedPayloadPairs" : [ { "index" : 0, "requestIndex" : 1, "signedUploadURL" : "https://signed-url" } ] }
```


### List Workers

-   GET /api/v1/workers/

#### Path parameters

No parameters


#### Query parameters

| Parameter | Type    | Optional | Description           |
|-----------|---------|----------|-----------------------|
| page      | Integer | true     | Default value: '0'\.  |
| size      | Integer | true     | Default value: '50'\. |


#### Request fields

No request body


#### Response fields

| Path                        | Type            | Optional | Description |
|-----------------------------|-----------------|----------|-------------|
| content                     | Array\[Object\] | true     |             |
| content\[\]\.lastModified   | String          | true     |             |
| content\[\]\.totalProcessed | Integer         | true     |             |
| content\[\]\.name           | String          | true     |             |
| content\[\]\.state          | String          | true     |             |
| content\[\]\.owner          | String          | true     |             |
| content\[\]\.id             | String          | true     |             |
| page                        | Object          | true     |             |
| page\.size                  | Integer         | true     |             |
| page\.totalElements         | Integer         | true     |             |
| page\.totalPages            | Integer         | true     |             |


#### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/workers/' -i -X GET

#### Example response

HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Content-Length: 848

```
{ "_embedded" : { "workerResourceList" : [ { "lastModified" : "2019-09-18T08:38:02.514370Z", "totalProcessed" : 0, "name" : "Media Encoder Cluster Worker 1", "state" : "IDLE", "owner" : "orgid@AdobeOrg", "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/workers/f845dca5-19bf-4939-8e1c-d891080e3ed0" } }, "id" : "f845dca5-19bf-4939-8e1c-d891080e3ed0" } ] }, "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/workers/?page=0&size=50&sort=lastModified,desc" }, "byId" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/workers{/id}", "templated" : true } }, "page" : { "size" : 50, "totalElements" : 1, "totalPages" : 1, "number" : 0 } }
```


### Delete Worker 
-   DELETE /api/v1/workers/{id}

#### Path parameters

| Parameter | Type   | Optional | Description |
|-----------|--------|----------|-------------|
| id        | Object | false    |             |

#### Query parameters

No parameters


#### Request fields

No request body


#### Response fields

| Path           | Type    | Optional | Description |
|----------------|---------|----------|-------------|
| lastModified   | String  | true     |             |
| totalProcessed | Integer | true     |             |
| name           | String  | true     |             |
| state          | String  | true     |             |
| owner          | String  | true     |             |
| id             | String  | true     |             |

#### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/workers/f895c6d6-4709-45b7-babe-06d93ff573a5' -i -X DELETE \
    -H 'X-Api-Key: your-api-key' \
    -H 'Authorization: Bearer the-access-token' \
    -H 'Content-Type: application/json;charset=UTF-8'

#### Example response

HTTP/1.1 204 No Content



