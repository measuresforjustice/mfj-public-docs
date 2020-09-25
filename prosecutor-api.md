## DRAFT - Prosecutor Data Delivery REST API

The data delivery API is a REST service for the receipt of ongoing prosecutor data by MFJ. All JSON is UTF-8. The service will be HTTPS-only.

The service uses standard HTTP response status codes:

* 200s for responses where no client or server errors occurred (though API errors are still possible and specified in the result)
* 300s for redirects.
* 400s for client errors (errors where resending the same request would not be successful). In general, API-specific errors will return a "400 - Bad Request" code with a JSON body that provides more specific error information.
* 500s for server errors (errors where resending the same request might be successful, because the error happened before the request was actually processed by the service).


## Authentication

The API is stateless/sessionless. All requests to the system will contain a basic auth element in the request header:

```
Authentication: Basic {base 64 encoded username:password (separated by colon)}
```

So if the username were mel and the password were 12345, the header element would be:

```
Authentication: Basic bWVsOjEyMzQ1
```

Requests without the header or with invalid credentials will receive a 401 response code from the server.

## File Upload

File upload is a three-step process The client:

1. begins (or resumes) an upload with an `api/v1/upload/start` request
2. uploads the file in parts using one or more `api/v1/upload/part` requests
3. finalizes the upload with an `api/v1/upload/complete` request

Uploads that were begun but not completed will remain available to resume for 96 hours from the initial `start` request.

These requests are documented below.

### "start" request

```
POST /api/v1/upload/start?id={upload ID}
```

This request begins or resumes a file upload.

`upload ID` is a client-supplied string representing the individual upload. It must be between 1 and 100 characters in length (inclusive) and contain only ASCII alphanumeric characters, underscores and dashes. Using a client-supplied ID allows the client to control whether a request is to initiate a new upload or to continue an incomplete one. IDs are non-reusable; after an upload is complete, a subsequent call to "start" with the same ID will fail.

The server will respond with a JSON object in the following form:

```
{
  "action": "start",
  "id": {file upload ID that was provided in the request},
  "code": {error code or zero for success},
  "message": {descriptive error message - blank for success},
  "parts": [{array of existing part numbers - empty if new upload, null if complete in progress}]
}

```

Values for `code` are:

* 0 - no error (successful request)
* 2 - complete called for this ID, not finished verifying
* 1000 - invalid ID
* 1020 - ID already used

Sending start requests in the middle of sending part requests is valid, and basically acts as a way to retrieve the status of an upload (in order to resume it or for another reason).

For any response with a code of zero, the HTTP response code will be 200. For any response with a code of 2, the HTTP response code will be 202. For any other (error) code, the HTTP response code will be 400.

### "part" request

```
POST /api/v1/upload/part?id={upload ID}&partNo={part number}&partSize={size in bytes of part}
```

TThis request sends a file upload part. The content-type of the request is application/octet-stream. The payload contains a portion of the file of the specified size.

The query params are:

* `id` - the client-supplied upload ID to which this file part belongs
* `partNo` - a number between 0 and 9999 (inclusive) that indicates the part’s position relative to other parts.
* `partSize` - the size in bytes of the payload

The server will respond with a JSON object in the following form:

```
{
  "action": "part",
  "id": {file upload ID that was provided in request},
  "partNo": {partNo that was provided in request},
  "partSize": {partSize that was provided in request},
  "code": {error code or zero for success},
  "message": {descriptive error message - blank for success}
}
```

Values for `code` are:

* 0 - no error (successful request)
* 1000 - invalid ID
* 1010 - no upload started with this ID
* 1300 - invalid part number
* 1400 - invalid part size
* 1500 - partSize does not match the size of the payload

Limits on part uploads:

* minimum part size is 2Mb for all but the final part
* maximum number of parts is 10,000

Parts can be sent in any order. Parts can be sent more than once (the most recent part is kept). An error response to a part request means only that the server does not count that part as having been received.

For any response with a code of zero, the HTTP response code will be 200. For any other (error) code, the HTTP response code will be 400.

### "complete" request

```
POST /api/v1/upload/complete
```

This request completes a file upload. The body of the request is a JSON object of the following form:

```
{
  "id": {client-supplied upload ID},
  "fileSize": {size in bytes of the entire file},
  "checksum": {md5 checksum of the entire file},
  "mimeType": {mime type},
  "stateCode": {state code},
  "location": {location},
  "countyName": {county name}
}
```

`id` is the client-supplied string representing the upload to be completed.

`fileSize` is the size of the entire uploaded file in bytes.

`checksum` is an md5 sum for the entire uploaded file.

`mimeType` specifies the format of the data that will be uploaded. Valid values are:

* `text/csv`
* `application/json`
* `application/xml`

`stateCode` is the 2-letter state code.

`locationCode` is the location code.

`countyName` is the county/borough/parish/judicial district name.

The server will respond with a JSON object in the following form:

```
{
  "action": "complete",
  "id": {file upload ID that was provided in request},
  "fileSize": {size in bytes that was provided in the request},
  "checksum": {md5 checksum of the entire file},
  "code": {error code or zero for success},
  "message": {descriptive error message - blank for success}
}
```

Values for `code` are:

* 0 - no error (successful request)
* 2 - not finished verifying
* 1000 - invalid ID
* 1010 - no upload started with this ID
* 1030 - uploaded completed with this ID, but file size or checksum does not match
* 1600 - parts are missing
* 1700 - file size does not match
* 1800 - checksum does not match
* 2000 - file structure is invalid
* 2100 - file schema is invalid
* 2200 - file contents are invalid


Error codes below 2000 do not invalidate the upload, and the upload remains available for completion.

Because file verification can take some time, the server will respond with an HTTP response code of 202 if it was unable to complete verification within 10 seconds of receiving the request. In this case the code in the response JSON payload will be 2. The client can re-send the same request to get a final status for the upload.

For any response with a code of zero, the HTTP response code will be 200. For any response with a code of 2, the HTTP response code will be 202. For any other (error) code, the HTTP response code will be 400.

Uploads must be completed within one week of being started.
Uncompleted uploads will be deleted, and their IDs may be reused.

## Upload File Structure and Format

The following fields will be present in the uploaded file:


| Field Name | Data Type | Values |
|---|---|---|
| County | string | County name |
| FileNumber | string | ###-###### |
| Status | string | e.g. OPEN, CLOSED |
| ReferralDate | date | ISO-8601 date |
| ArrestDate | date | ISO-8601 date |
| RefAgency | string | |
| Municipality | string | |
| AgencyCaseNum | string | |
| Unit | string | |
| DefendantState | string | e.g. MO, PA, IL |
| DefendantRace | string | e..g. W, B, I, A, H, U |
| DefendantGender | string | e..g. M, F, U |
| DefendantSID | string | |
| PersonID | integer |  |
| CourtCaseNum | string |  |
| IncidentDate | date | ISO-8601 date |
| CountNumber | integer |  |
| LeadChargeFlag | string | e.g., Y, N |
| ReferralCharge | string |  |
| ReferralStatute | string |  |
| ReferralChargeDescription | string |  |
| ReferralSeverity | string | e.g. F, M, I, U, L |
| ReferralClass | string | e.g. A, B, C, D, E, U |
| ReferralModifier | string | e.g. Attempted, Conspiracy |
| ReferralNCIC | string | NCIC is numeric but uses leading zeros |
| ReferralNCICEnhancerDesc | string |  |
| ChargeCode | string |  |
| ChargeStatute | string |  |
| ChargeDescription | string |  |
| Severity | string | e.g., F, M, I, L |
| Class | string | e.g., A. B, C, D, U |
| ChargeModifier | string | e.g., Attempted, Conspiracy |
| ChargeNCIC | string | NCIC is numeric but uses leading zeros |
| ChargeNCICEnhancerDesc | string |  |
| CaseScreeningDecision | string | e.g., Accepted, Declined |
| CaseScreeningDate_ReviewOfCharges | date | ISO-8601 date |
| IssuedDate | date | ISO-8601 date |
| ChargeDispo | string |   |
| DispoDate | date | ISO-8601 date |
| SentenceDate | date | ISO-8601 date |
| CaseVicCount | integer |  |
| VictimRace | string | e..g. W, B, I, A, H, U |
| VictimGender | string | e.g. M, F, U |
| AgeAtOffenseDate | integer |  |
| Domestic | string | e.g. Yes, No |
| CaseIssuedToDispDays | integer |  |
| CaseIssuedToSentDays | integer |  |

Data will be accepted in any of three formats: CSV, JSON or XML.

CSV files will use the field names above as headers on the first row, and the file will conform to RFC-4180.

JSON files will be an array of objects, one object per row, using the field names above as keys. The file will conform to RFC-7159.

XML files will be a series of Record elements contained within a root Records element (TODO: possibly better names?). Each Record element will contain a row of data, using the above field names as element names (e.g. <Status>DISPOSED</Status>). The file will conform to the W3C XML 1.0 Specification.

## Configuration

The default location is `~/.config/file-upload-service/config.properties`.
The location can be overriden with env var `$FILE_UPLOAD_SERVICE_CONFIG`.

The file is a java properties file, containing the following properties:

  * `interface` - bind interface. Default: `127.0.0.1`
    * In docker, you probably want to set this to `0.0.0.0`.
  * `port` - bind port. Default: `4567`
  * `ldap.uri` - LDAP URI. Always uses StartTLS. Default: `ldap://ldap.mfj.io`
  * `authz.group` - LDAP group that user must have to use the service.
  * `aws.region` - AWS region
  * `aws.profile` - AWS profile. Default: default
  * `s3.bucket` - Name of S3 bucket that data is stored in.
  * `s3.prefix` - S3 key prefix for all stored data. Default: none
  * `lock-table` - Name of DynamoDB lock table.

## Health Endpoint

GET `http://{host}:{port}/api/v1/.health` returns 200 and the text "healthy".

## Docker

Mount the configuration file and set env var `$FILE_UPLOAD_SERVICE_CONFIG`.

Java options can be set by setting env vars `$JAVA_OPTS.`

## Shutdown

On SIGTERM, it will spend up to 30 seconds responding to already started requests.

## Scaling

All calls are stateless. DynamoDB is used for locking, data is stored in S3.
Multiple copies of the web service may be run simultaneously.

