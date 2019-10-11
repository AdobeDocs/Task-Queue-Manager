# Adobe Task Queue Manager getting started guide

## Introduction

This guide describes how to setup a system to allow access to the cloud API of the Adobe Task Queue Manager and perform a set of basic requests.

## Features

Task Queue Manager allows customers to send jobs from one machine to another while receiving real time updates about progress. These jobs can either
be encode jobs (similar to Adobe Media Encoder Remote) or script jobs with arbitrary extension code. The transfer and delivery from the client to the
server machine is handled by queues in the cloud. A server/worker can be deployed to any computer running PremierePro or Adobe Media Encoder. The
content of e.g. encode jobs is composed of the project file (or team project snapshot) an optional export preset file and meta data.

## Installing a worker

```
Install Java jdk 11
Install Premiere Pro 14.0 and Adobe Media Encoder 14.0 (2020)
The worker can be installed by extracting the zip file to any folder on a machine. It can also be setup to run as a service or started as a process.
Retrieve the account credentials from console.adobe.io and configure the cc-worker.xml file
Make sure there is a queue already created to be set on the worker (created via API or from Premiere Pro DOM calls)
```
## Account setup

Login using an administrator account of your domain onhttps://console.adobe.io/. And choose to create a new integration.


On the following screen choose "Access an API"

Choose "Task Queue Manager" from the list.

In the following screen you can name the integration and provide a security certificate used for authentication - see the "Creating the worker keystore file
and certificate" chapter in the developer guide below or

https://www.adobe.io/authentication/auth-methods.html#!AdobeDocs/adobeio-auth/master/AuthenticationOverview/ServiceAccountIntegration.md#step-2-
create-a-public-key-certificate. Drag the "certificate_pub.crt" into the drop box and continue.


The integration is now created and you can use the following page to retrieve all configuration values needed to setup a worker. (API_ADOBE_USER_ID
in the config file refers to theTechnical account ID on this page)

### Running as a process (for debugging only)

Update the run-as-process.bat file found in the worker directory with the information found on the adobe.io developer console. Then start the bat file. Make
sure java RE >= 11 is installed and available on the PATH.

### Running as a service

The configuration file cc-worker.xml can be found in the service folder. The XML lists all required values which were generated in the previous step for
creating a JWT token on Adobe.io. Make sure to use a local user which has sufficient rights to start both Premiere Pro and Adobe Media Encoder. The
service should not be run on the default service account.


```
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
```
The process can be setup to run as a service as soon as the configuration file and keystore file are correctly setup:

This requires Admin rights on the local machine

```
cc-worker.exe install
```
```
cc-worker.exe start
```
The process will populate log files to the /logs directory of the same folder. Make sure the process is running as a user which is allowed to write to that
folder, and has rights to start Adobe Media Encoder. The correct user can be verified when opening the service list, opening the "Adobe Task Queue
Manager" service and inspecting the "log on" tag. It should show the service using a specific user account instead of "Local system account".

Please note cc-worker.exe install has to be repeated every time the user password is changed in the configuration.

## Developer guide

### Generating an API token

For testing purpose you can use the Premiere Pro scripting engine to request a token for the currently logged in user. In production this workflow will be
replaced with:https://www.adobe.io/apis/cloudplatform/console/authentication/createjwt.html. The integration on Adobe.Io can be created via:https://console
.adobe.io/integrations.

In the ExtendScript Toolkit connect to Premiere Pro and invoke the following command:

```
app.enableQE();
```
```
taskQueueManagerObj = qe.tqm.logIntoTaskQueueManager("");
```
```
var authTokenObj = qe.tqm.getAuthToken();
var bearerToken = authTokenObj.token;
```
### Accessing the API

The API can be accessed via the URL: https://cloud-dispatcher-beta.adobe.io

The API uses the JSON+HAL format (https://en.wikipedia.org/wiki/Hypertext_Application_Language).


It requires the "Authorization" header to be set to "Bearer yourBearerTokenFromExtendScriptHere" and the "X-Api-Key" header set to "task-queue-
manager-beta-1".

### Using the Task Queue Manager API from within Premiere Pro (DOM API)

This example script will lookup a queue from the list of existing queues (without creating a new one). A typical workflow would involve one central queue
being created by an administrator (assigned to the organisation) which all users can share for submitting work to the central pool of workers.

```
qe.tqm.logIntoTaskQueueManager("");
var queueId = "same-queue-id-as-used-in-the-worker";
var queueList = qe.tqm.getQueueListObject();
queueList.loadSync();
var queue = null;
var queues = queueList.getQueues();
```
```
for (var i=0; i < queues.length; i++) {
if (queues[i].identifier == queueId) {
queue = queues[i];
break;
}
}
```
```
var job = queue.createRemoteAMERenderJob(
"example encode job 1",
"c:\\full\\path\\to\\team\\projects\\snapshot\\tp-snapshot.tp2snap",
"c:\\full\\path\\to\\team\\projects\\snapshot\\ExportPresetH264MatchSequence.epr",
"c:\\worker\\harddrive\\or\\network\\share\\output.mp4"
);
var result = job.loadSync();
```
### API

All API calls are scoped to the qe dom.

#### Root Level Methods

##### qe.tqm.logIntoTaskQueueManager(callback-method-name) - returns Boolean (success)

used to create an authentication token from the currently logged in user to communicate the the TQM API

##### qe.tqm.loginByRegion(regionName,"") -returns Boolean (success)

used to create an authentication token from the currently logged in user on the region provided as string parameter to communicate the the TQM API

##### qe.tqm.getActiveRegion(); - returns String

returns the active region user is currently logged in.

##### qe.tqm.getQueueListObject() - returns QueueListObject

creates a request object to read the list of queues visible to the logged in user

##### qe.tqm.createQueue(queueName, ownerId) returns QueueObject

create a request with the given parameters for creating a new queue, owned by the user (if a blank ownerId is given) or the organisation of the ownerId

##### qe.tqm.getAuthToken() - returns AuthenticationObject

returns the Authentication object which can be then used to get current authentication token usingAuthenticationObject.token(only available after calling
the logIntoTaskQueueManager method)

##### qe.tqm.setDiscoveryURL(String) - return Boolean (success)

sets the base discovery URL used to communicating with the api (can e.g. be used to switch to a different region or localhost for debugging)

#### Queue List Level Methods


##### queueList.getQueues() returns an array of Queue objects

after the request was loaded (async/sync) the list of queues is available in this array

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

creates a project file based job and attaches both the export preset and the project to the request object.

##### queue.prepareRemoteAMERenderProjectJob(jobName, localProjectFilePath, itemGUID, localExportPresetPath,

##### remoteOutputFilePath) - returns a Job request object

createsproject file based job that will not be automatically be started and attaches both the export preset and the project to the request object.

##### queue.createRemoteAMERenderJob(jobName, localTeamProjectSnapshotPath, localExportPresetPath, remoteOutputFilePath) -

##### returns a Job request object

creates a team projects snapshot based job and attaches both the snapshot and export preset to the request object.

##### queue.prepareRemoteAMERenderJob(jobName, localTeamProjectSnapshotPath, localExportPresetPath,

##### remoteOutputFilePath) - returns a Job request object

creates a team projects snapshot based job that will not be automatically be started and attaches both the snapshot and export preset to the request object.

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

##### the estimated completion time inthe 'YYYY-MM-DDTHH:MM:SS.XXXZ'format.


job.lastModified- returns a String lastModified property

the last modification timestamp of the job in the'YYYY-MM-DDTHH:MM:SS.XXXZ'format.

##### job.estimatedCompletion- returns a String estimatedCompletion property

the estimated-completion timestamp of the job in the'YYYY-MM-DDTHH:MM:SS.XXXZ' format.

##### job.state- returns a String state property

the current state of the job (i.e WAITING, PROGRESSING, COMPLETED, CANCELED)

##### job.type- returns a String type property

the type of the job. (i.e AME.2020.encode, AE.2020.render, PPRO.2020.ingest, etc)

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

adds an input file the the job - (e.g. if the job is created via prepareRemoteAMERenderJob - the job will not be started and may receive additional input
files, and can then be started via the job.start() call)

##### job.start() - returns a Boolean (success)

start the job

##### job.cancel() - returns a Boolean (success)

cancel the job

##### job.loadSync()- returns a job id(success)

process the request sync (blocking)

job.loadSync(job.identifier) - take job id as string input andreturns a job id(success) and refreshes the property of job passed as input

process the request sync (blocking)

##### job.loadAsync()- returns a Boolean (success)

process the request async (non-blocking)

#### Additional Snapshot Methods

##### app.project.rootItem.children[childNumber].saveProjectSnapshot(snapShotPath) - take childNumber as integer, snapShotPath as string and returns a Boolean (success)

##### app.anywhere.openTeamProjectSnapshot(snapShotPath) - take snapShotPath as string and returns a Boolean (success)

##### project.applyProjectSnapshot(snapShotPath, keepSessionProperties,deleteAssetsNotInSnapshot) - take snapShotPath as string, keepSessionProperties as boolean,  deleteAssetsNotInSnapshot as boolean returns a Boolean (success)

The snapshot contents will be merged into the currently-open Team Project. If keepSessionProperties is set to true, the existing properties will not be replaced; otherwise, the project name and description will be replaced by those of the snapshot. If deleteAssetsNotInSnapshot is set to true, any assets not in the snapshot will be deleted; otherwise, the snapshot will be merged into the existing project (snapshot assets will overwrite any existing identical assets).

## Adobe Task Queue Manager API documentation

## Table of Contents

```
Introduction
Authentication
Creating the worker keystore file and certificate
Resources
```

Discovery Links
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
List My Jobs
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Input
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
List Workers
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Delete Worker
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
List Queues
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Find Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Delete Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
List Jobs By Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Listen For Queue Job Updates
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Delete Job
Path parameters


```
Query parameters
Request fields
Response fields
Example request
Example response
Create Team Project Encode Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Project File Encode Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Premiere Pro Script Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Premiere Pro Ingest Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Premiere Pro OMF Conversion Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Create Premiere Pro AAF Conversion Job On Queue
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
Patch Job
Path parameters
Query parameters
Request fields
Response fields
Example request
Example response
```
## Introduction

## Getting started with the Adobe Task Queue Manager API. The API uses the

## HAL JSON format:http://stateless.co/hal_specification.html. The API can be

## navigated by following the links starting from the discovery endpoint.

### Authentication

The API supports authentication through a bearer token header and an api key. This can be generated via a JWT token workflow using the adobe.io
integration:https://www.adobe.io/apis/cloudplatform/console/authentication/gettingstarted.html. Setting both the 'Authorization' and the 'X-Api-Key' header
is required for every request.

#### Creating the worker keystore file and certificate


openssl req -nodes -text -x509 -newkey rsa:2048 -keyout secret.pem -out certificate.pem -days 356

openssl pkcs8 -topk8 -inform PEM -outform DER -in secret.pem -nocrypt > secret.key

openssl pkcs12 -export -in certificate.pem -inkey secret.pem -out my.p

keytool -importkeystore \
-deststorepass 123456 -destkeypass 123456 -destkeystore keystore.ks \
-srckeystore my.p12 -srcstoretype PKCS12 -srcstorepass 123456

keytool -changealias -keystore keystore.ks -alias 1 -destalias AdobePrivateKey

## Resources

#### Discovery Links

###### GET /

Returns the list of all links available on the root API.

##### Path parameters

No parameters.

##### Query parameters

No parameters.

##### Request fields

No request body.

##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/' -i -X GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 760

{ "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io" }, "jobs" : { "href" : "https://cloud-
dispatcher-beta.adobe.io/api/v1/jobs/{?page,size}", "templated" : true }, "queues" : { "href" : "https://cloud-
dispatcher-beta.adobe.io/api/v1/queues/{?page,size}", "templated" : true }, "workers" : { "href" : "https://cloud-
dispatcher-beta.adobe.io/api/v1/workers/{?page,size}", "templated" : true }, "region-ue1" : { "href" : "https://cl
oud-dispatcher-beta-ue1.adobe.io" }, "region-ew1" : { "href" : "https://cloud-dispatcher-beta-ew1.adobe.io" }, "re
gion-an1" : { "href" : "https://cloud-dispatcher-beta-an1.adobe.io" } } }

#### List My Jobs

GET /api/v1/jobs/

##### Path parameters

No parameters.

##### Query parameters

```
Parameter Type Optional Description
```
```
page Integer true Default value: '0'.
```
```
size Integer true Default value: '50'.
```
##### Request fields


No request body.

##### Response fields

```
Path Type Optional Description
```
```
content Array[Object] true
```
```
content[].lastModified String false
```
```
content[].name String true
```
```
content[].state String false
```
```
content[].type String false
```
```
content[].lockId String true
```
```
content[].progress Decimal true
```
```
content[].progressState String true
```
```
content[].estimatedCompletion String true
```
```
content[].created String false
```
```
content[].id String false
```
```
page Object true
```
```
page.size Integer true
```
```
page.totalElements Integer true
```
```
page.totalPages Integer true
```
```
page.number Integer true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/' -i -X GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Length: 1891
Content-Type: application/hal+json;charset=UTF-

{ "_embedded" : { "jobResourceList" : [ { "lastModified" : "2019-06-05T11:11:24.772705Z", "name" : "Encode
Project to MP4", "state" : "WAITING", "type" : "AME.2020.encode", "lockId" : "workerId", "progress" : 0.0, "progre
ssState" : null, "estimatedCompletion" : null, "created" : "2019-06-05T11:11:24.772698Z", "_links" : { "self" : {
"href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/c7b9e3ad-f029-4ec1-9cbd-87509e15fa0d" }, "inputs" :
{ "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/c7b9e3ad-f029-4ec1-9cbd-87509e15fa0d/inputs" }, "in
puts-presign" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/c7b9e3ad-f029-4ec1-9cbd-
87509e15fa0d/inputs/presign" }, "results" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs
/c7b9e3ad-f029-4ec1-9cbd-87509e15fa0d/results" }, "results-presign" : { "href" : "https://cloud-dispatcher-beta.
adobe.io/api/v1/jobs/c7b9e3ad-f029-4ec1-9cbd-87509e15fa0d/results/presign" }, "waitForCancel" : { "href" : "https:
//cloud-dispatcher-beta.adobe.io/api/v1/jobs/c7b9e3ad-f029-4ec1-9cbd-87509e15fa0d/waitFor/canceled" } }, "id" : "c
7b9e3ad-f029-4ec1-9cbd-87509e15fa0d" } ] }, "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.
io/api/v1/jobs/?page=0&size=50&sort=lastModified,desc" }, "byId" : { "href" : "https://cloud-dispatcher-beta.
adobe.io/api/v1/jobs/{id}", "templated" : true }, "updates" : { "href" : "https://cloud-dispatcher-beta.adobe.io
/api/v1/jobs/since/2019-06-05T11:11:24.772705Z/" } }, "page" : { "size" : 50 , "totalElements" : 1 , "totalPages" : 1
, "number" : 0 } }

#### Create Input

POST /api/v1/jobs/{id}/inputs/presign

##### Path parameters

```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.


##### Request fields

```
Path Type Optional Description
```
```
payloads Array[Object] true
```
```
payloads[].index Integer true
```
```
payloads[].length Integer true
```
```
payloads[].name String true
```
##### Response fields

```
Path Type Optional Description
```
```
signedPayloadPairs Array[Object] true
```
```
signedPayloadPairs[].index Integer true
```
```
signedPayloadPairs[].requestIndex Integer true
```
```
signedPayloadPairs[].signedUploadURL String true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/06ec363b-ab6d-481c-a0a0-0ba3f6204e6f/inputs/presign' -
i -X POST \
-H 'Content-Type: application/json' \
-d '{"payloads":[{"index":1,"length":123,"name":"input.prproj"}]}'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/json;charset=UTF-
Content-Length: 124

{ "signedPayloadPairs" : [ { "index" : 0 , "requestIndex" : 1 , "signedUploadURL" : "https://signed-url" } ] }

#### List Workers

GET /api/v1/workers/

##### Path parameters

No parameters.

##### Query parameters

```
Parameter Type Optional Description
```
```
page Integer true Default value: '0'.
```
```
size Integer true Default value: '50'.
```
##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
content Array[Object] true
```
```
content[].lastModified String true
```
```
content[].name String true
```
```
content[].state String true
```
```
content[].owner String true
```
```
content[].totalProcessed Integer true
```
```
content[].id String true
```
```
page Object true
```

```
page.size Integer true
```
```
page.totalElements Integer true
```
```
page.totalPages Integer true
```
```
page.number Integer true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/workers/' -i -X GET

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 848

{ "_embedded" : { "workerResourceList" : [ { "lastModified" : "2019-06-05T11:11:28.526920Z", "name" : "Media
Encoder Cluster Worker 1", "state" : "IDLE", "owner" : "orgid@AdobeOrg", "totalProcessed" : 0 , "_links" : { "self"
: { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/workers/b0dcc20b-3417-45af-a044-911d5ce85726" } }, "i
d" : "b0dcc20b-3417-45af-a044-911d5ce85726" } ] }, "_links" : { "self" : { "href" : "https://cloud-dispatcher-
beta.adobe.io/api/v1/workers/?page=0&size=50&sort=lastModified,desc" }, "byId" : { "href" : "https://cloud-
dispatcher-beta.adobe.io/api/v1/workers{/id}", "templated" : true } }, "page" : { "size" : 50 , "totalElements" : 1
, "totalPages" : 1 , "number" : 0 } }

#### Delete Worker

DELETE /api/v1/workers/{id}

Deletes the worker activity data.

##### Path parameters

```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.

##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
lastModified String true
```
```
name String true
```
```
state String true
```
```
owner String true
```
```
totalProcessed Integer true
```
```
id String true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/workers/d95df33c-917c-41b0-8d17-0e8b36fa2b1c' -i -X DELETE \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token' \
-H 'Content-Type: application/json;charset=UTF-8'

##### Example response

HTTP/1.1 204 No Content

#### List Queues

GET /api/v1/queues


##### Path parameters

No parameters.

##### Query parameters

```
Parameter Type Optional Description
```
```
page Integer true Default value: '0'.
```
```
size Integer true Default value: '50'.
```
##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
content Array[Object] true
```
```
content[].name String true
```
```
content[].owner String false
```
```
content[].id String false
```
```
page Object true
```
```
page.size Integer true
```
```
page.totalElements Integer true
```
```
page.totalPages Integer true
```
```
page.number Integer true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues' -i -X GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 1289

{ "_embedded" : { "queueResourceList" : [ { "name" : "Media Encoder Queue Hamburg", "owner" : "example ORG", "_lin
ks" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/e6fcd96d-ed43-4034-a7d4-
b2867e7cd694" }, "listen" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/e6fcd96d-ed43-4034-
a7d4-b2867e7cd694/listen" }, "progressingBy" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues
/e6fcd96d-ed43-4034-a7d4-b2867e7cd694/jobs/progressingBy/{lockId}{?page,size}", "templated" : true }, "jobs" : { "
href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/e6fcd96d-ed43-4034-a7d4-b2867e7cd694/jobs{?page,
size}", "templated" : true } }, "id" : "e6fcd96d-ed43-4034-a7d4-b2867e7cd694" } ] }, "_links" : { "self" : { "href"
: "https://cloud-dispatcher-beta.adobe.io/api/v1/queues?page=0&size=50&sort=lastModified,desc" }, "byId" : { "hre
f" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues{/id}", "templated" : true } }, "page" : { "size" : 50 ,
"totalElements" : 1 , "totalPages" : 1 , "number" : 0 } }

#### Find Queue

GET /api/v1/queues/{id}

##### Path parameters

```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.

##### Request fields


No request body.

##### Response fields

```
Path Type Optional Description
```
```
name String true
```
```
owner String false
```
```
id String false
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/910a1e30-f528-4fde-b9e3-66a92b129979' -i -X GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 782

{ "name" : "Media Encoder Queue Hamburg", "owner" : "example ORG", "_links" : { "self" : { "href" : "https://cloud
-dispatcher-beta.adobe.io/api/v1/queues/910a1e30-f528-4fde-b9e3-66a92b129979" }, "listen" : { "href" : "https://cl
oud-dispatcher-beta.adobe.io/api/v1/queues/910a1e30-f528-4fde-b9e3-66a92b129979/listen" }, "progressingBy" : { "hr
ef" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/910a1e30-f528-4fde-b9e3-66a92b129979/jobs
/progressingBy/{lockId}{?page,size}", "templated" : true }, "jobs" : { "href" : "https://cloud-dispatcher-beta.
adobe.io/api/v1/queues/910a1e30-f528-4fde-b9e3-66a92b129979/jobs{?page,size}", "templated" : true } }, "id" : "
a1e30-f528-4fde-b9e3-66a92b129979" }

#### Create Queue

POST /api/v1/queues

##### Path parameters

No parameters.

##### Query parameters

No parameters.

##### Request fields

```
Path Type Optional Description
```
```
name String false
```
```
owner String true
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues' -i -X POST \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token' \
-H 'Content-Type: application/json' \
-d '{"name":"Media Encoder Queue Munich","owner":"orgId@AdobeOrg"}'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/queues/ddd25dc8-1a53-4446-a1ee-266bbded0d

#### Delete Queue

DELETE /api/v1/queues/{id}

##### Path parameters


```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.

##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
name String true
```
```
owner String false
```
```
id String false
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/284c87b2-131d-4a81-a9c3-106de0ed9d28' -i -X DELETE \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token' \
-H 'Content-Type: application/json' \
-d '{"name":"Media Encoder Queue Munich","owner":"orgId@AdobeOrg"}'

##### Example response

HTTP/1.1 204 No Content

#### List Jobs By Queue

GET /api/v1/queues/{queueId}/jobs/

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

```
Parameter Type Optional Description
```
```
page Integer true Default value: '0'.
```
```
size Integer true Default value: '50'.
```
##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
content Array[Object] true
```
```
content[].lastModified String false
```
```
content[].name String true
```
```
content[].state String false
```
```
content[].type String false
```
```
content[].lockId String true
```
```
content[].progress Decimal true
```

```
content[].progressState String true
```
```
content[].estimatedCompletion String true
```
```
content[].created String false
```
```
content[].id String false
```
```
page Object true
```
```
page.size Integer true
```
```
page.totalElements Integer true
```
```
page.totalPages Integer true
```
```
page.number Integer true
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/bc952bf5-de73-47a5-b6e5-2c4f87af2bc9/jobs/' -i -X
GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 1864

{ "_embedded" : { "jobResourceList" : [ { "lastModified" : "2019-06-05T11:11:24.848718Z", "name" : "Encode
Project to MP4", "state" : "WAITING", "type" : "AME.2020.encode", "lockId" : "workerId", "progress" : 0.0, "progre
ssState" : null, "estimatedCompletion" : null, "created" : "2019-06-05T11:11:24.848713Z", "_links" : { "self" : {
"href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/d32e1542-917b-433d-a5c0-4a0dcb75bfc9" }, "inputs" :
{ "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/d32e1542-917b-433d-a5c0-4a0dcb75bfc9/inputs" }, "in
puts-presign" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/d32e1542-917b-433d-a5c0-
4a0dcb75bfc9/inputs/presign" }, "results" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs
/d32e1542-917b-433d-a5c0-4a0dcb75bfc9/results" }, "results-presign" : { "href" : "https://cloud-dispatcher-beta.
adobe.io/api/v1/jobs/d32e1542-917b-433d-a5c0-4a0dcb75bfc9/results/presign" }, "waitForCancel" : { "href" : "https:
//cloud-dispatcher-beta.adobe.io/api/v1/jobs/d32e1542-917b-433d-a5c0-4a0dcb75bfc9/waitFor/canceled" } }, "id" : "d
32e1542-917b-433d-a5c0-4a0dcb75bfc9" } ] }, "_links" : { "self" : { "href" : "https://cloud-dispatcher-beta.adobe.
io/api/v1/queues/bc952bf5-de73-47a5-b6e5-2c4f87af2bc9/jobs/?page=0&size=50&sort=lastModified,desc" }, "queue-
updates" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/queues/bc952bf5-de73-47a5-b6e5-2c4f87af2bc
/jobs/since/2019-06-05T11:11:24.853360Z/" } }, "page" : { "size" : 50 , "totalElements" : 1 , "totalPages" : 1 , "num
ber" : 0 } }

#### Listen For Queue Job Updates

GET /api/v1/queues/{queueId}/jobs/since/{since}/

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
```
since Object false
```
##### Query parameters

No parameters.

##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
result Object true
```
```
setOrExpired Boolean true
```
##### Example request


$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/978978f1-0e11-4c6d-8e59-dfa388d50e14/jobs/since/2019-
06-05T11:11:24.287887Z/' -i -X GET \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token'

##### Example response

###### HTTP/1.1 200 OK

Content-Type: application/hal+json;charset=UTF-
Content-Length: 1592

{ "_embedded" : { "jobResourceList" : [ { "lastModified" : "2019-06-05T11:11:24.287692Z", "name" : "Encode
Project to MP4", "state" : "WAITING", "type" : "AME.2020.encode", "lockId" : "workerId", "progress" : 0.0, "progre
ssState" : null, "estimatedCompletion" : null, "created" : "2019-06-05T11:11:24.287688Z", "_links" : { "self" : {
"href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/1921305f-451c-4f34-8963-4bd368abcecc" }, "inputs" :
{ "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/1921305f-451c-4f34-8963-4bd368abcecc/inputs" }, "in
puts-presign" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/1921305f-451c-4f34-8963-
4bd368abcecc/inputs/presign" }, "results" : { "href" : "https://cloud-dispatcher-beta.adobe.io/api/v1/jobs
/1921305f-451c-4f34-8963-4bd368abcecc/results" }, "results-presign" : { "href" : "https://cloud-dispatcher-beta.
adobe.io/api/v1/jobs/1921305f-451c-4f34-8963-4bd368abcecc/results/presign" }, "waitForCancel" : { "href" : "https:
//cloud-dispatcher-beta.adobe.io/api/v1/jobs/1921305f-451c-4f34-8963-4bd368abcecc/waitFor/canceled" } }, "id" : "
921305f-451c-4f34-8963-4bd368abcecc" } ] }, "_links" : { "queue-updates" : { "href" : "https://cloud-dispatcher-
beta.adobe.io/api/v1/queues/978978f1-0e11-4c6d-8e59-dfa388d50e14/jobs/since/2019-06-05T11:11:24.287692Z/" } } }

#### Delete Job

DELETE /api/v1/jobs/{id}

##### Path parameters

```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.

##### Request fields

No request body.

##### Response fields

```
Path Type Optional Description
```
```
lastModified String false
```
```
name String true
```
```
state String false
```
```
type String false
```
```
lockId String true
```
```
progress Decimal true
```
```
progressState String true
```
```
estimatedCompletion String true
```
```
created String false
```
```
id String false
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/07d190e7-c70b-4225-bc32-0724ff0b306d' -i -X DELETE \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token' \
-H 'Content-Type: application/json;charset=UTF-8'

##### Example response

HTTP/1.1 204 No Content


#### Create Team Project Encode Job On Queue

POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
jobParameters.js The parameters passed to the worker, specific to the job type
```
```
project.tp2snap The binary Team Projects snapshot
```
```
exportPreset.epr The binary export preset
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/7e366398-2fbb-4b99-a1dc-a88f159ba9f9/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"name":"Team Projects Export","type":"AME.2020.encode"}' \
-F 'jobParameters.js={"outputFile":"/worker/relative/output/directory/file.mp4","method":"
addTeamProjectsItemToBatch"}' \
-F 'project.tp2snap=<<project.tp2snap data>>' \
-F 'exportPreset.epr=<<exportPreset.epr data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/852ed4e8-88b2-4aa4-b5f0-c76f

#### Create Project File Encode Job On Queue

POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
jobParameters.js The parameters passed to the worker, specific to the job type
```
```
project.prproj The binary Project file
```

```
exportPreset.epr The binary export preset
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/49d2c431-9527-49b8-8332-f86010a1401f/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"name":"Project File Export","type":"AME.2020.encode"}' \
-F 'jobParameters.js={"outputFile":"/worker/relative/output/directory/file.mp4","method":"addDLToBatch","
itemGUID":"<optional sequenceId>"}' \
-F 'project.prproj=<<project.prproj data>>' \
-F 'exportPreset.epr=<<exportPreset.epr data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/76f9dfaf-edc9-4d45-9416-e2d6caf936ca

#### Create Premiere Pro Script Job On Queue

POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
script.jsx The extendscript code to execute on the attached project. The project is saved as a job result if the project was modified
```
```
project.prproj The binary Project file
```
```
exportPreset.epr The binary export preset
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/68e32bbf-fc36-4aa5-b63d-652935d8d3ec/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"name":"Premiere Pro Script","type":"PPRO.2020.script"}' \
-F 'script.jsx=<<script.jsx data>>' \
-F 'project.prproj=<<project.prproj data>>' \
-F 'exportPreset.epr=<<exportPreset.epr data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/06b348da-167d-4640-9624-74afb39cf5d

#### Create Premiere Pro Ingest Job On Queue


POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
jobParameters.js The parameters passed to the worker, specific to the job type
```
```
project.prproj The binary Project file
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/cc332a83-a299-44a9-89c8-838d5d716559/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"name":"Premiere Pro Ingest","type":"PPRO.2020.ingest"}' \
-F 'jobParameters.js={"files":["/path/to/ingest/file.mp4","/other/path/to/ingest/file.mov"]}' \
-F 'project.prproj=<<project.prproj data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/05306e14-8c27-490a-998e-d7f985b5700b

#### Create Premiere Pro OMF Conversion Job On Queue

POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
script.jsx The extendscript code to execute on the attached project. The project is saved as a job result if the project was modified
```
```
project.prproj The binary Project file
```
##### Response fields

No response body.


##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/c522ef55-c0c6-4795-9c17-5e37535f06c5/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"handleFramesOut":"if trim is 1, handle frames from 0 to 1000","outputFile":"\\\\a-network-
drive\\folder\\file","bitsPerSample":"16 or 24","encapsulation":true,"itemGUID":"3eb98659-d069-4df4-b820-
aa05b4bd9796","trimAudio":true,"name":"Premiere Pro Script","format":"WAV or AIFF","type":"PPRO.2020.omf-
conversion","title":"OMF Title","handeFramesIn":"if trim is 1, handle frames from 0 to 1000","sampleRate":"48000
or 96000"}' \
-F 'script.jsx=<<script.jsx data>>' \
-F 'project.prproj=<<project.prproj data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/67015ade-9210-45db-8d4c-cb430b6b0de8

#### Create Premiere Pro AAF Conversion Job On Queue

POST /api/v1/queues/{queueId}/jobs

##### Path parameters

```
Parameter Type Optional Description
```
```
queueId Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Part Description
```
```
jobDetails The job properties
```
```
script.jsx The extendscript code to execute on the attached project. The project is saved as a job result if the project was modified
```
```
project.prproj The binary Project file
```
##### Response fields

No response body.

##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/queues/147c25e2-9634-4786-b771-89e1d25377ca/jobs' -i -X
POST \
-H 'X-Api-Key: your-api-key' \
-H 'Content-Type: multipart/form-data' \
-H 'Authorization: Bearer the-access-token' \
-F 'jobDetails={"mixDownVideo":true,"format":"WAV or AIFF","embedAudio":true,"trimSources":true,"preset":"
optional preset","type":"PPRO.2020.aaf-conversion","sampleRate":"48000 or 96000","explodeToMono":true,"
outputFile":"\\\\a-network-drive\\folder\\file","bitsPerSample":"16 or 24","itemGUID":"cdd46bef-548b-445c-be25-
e88ec251b766","handleFrames":"number of 'handle' frames","name":"Premiere Pro AAF Conversion"}' \
-F 'script.jsx=<<script.jsx data>>' \
-F 'project.prproj=<<project.prproj data>>'

##### Example response

HTTP/1.1 201 Created
Location: https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/4a287e6a-b122-4ac3-be30-870899091da1

#### Patch Job

PATCH /api/v1/jobs/{id}

##### Path parameters


```
Parameter Type Optional Description
```
```
id Object false
```
##### Query parameters

No parameters.

##### Request fields

```
Path Type Optional Description
```
```
estimatedCompletion String true
```
```
progress Decimal true
```
```
progressState String true
```
```
state String true
```
##### Response fields

```
Path Type Optional Description
```
```
lastModified String false
```
```
name String true
```
```
state String false
```
```
type String false
```
```
lockId String true
```
```
progress Decimal true
```
```
progressState String true
```
```
estimatedCompletion String true
```
```
created String false
```
```
id String false
```
##### Example request

$ curl 'https://cloud-dispatcher-beta.adobe.io/api/v1/jobs/9d92eb0e-749d-4742-89b9-c61d93272550' -i -X PATCH \
-H 'X-Api-Key: your-api-key' \
-H 'Authorization: Bearer the-access-token' \
-H 'Content-Type: application/json;charset=UTF-8' \
-d '{"state":"CANCELED"}'

##### Example response

HTTP/1.1 204 No Content

## Last updated 2019-06-05 13:09:52 +0200


