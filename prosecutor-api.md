## Prosecutor Data Delivery REST API

The data delivery API is a REST service for the receipt of ongoing prosecutor data by MFJ. All JSON is UTF-8. The service will be HTTPS-only.

The service uses standard HTTP response status codes:

* 200s for responses where no client or server errors occurred (though API errors are still possible and specified in the result)
* 300s for redirects.
* 400s for client errors (errors where resending the same request would not be successful). In general, API-specific errors will return a "400 - Bad Request" code with a JSON body that provides more specific error information.
* 500s for server errors (errors where resending the same request might be successful, because the error happened before the request was actually processed by the service).

## Endpoint

The endpoint to access this API is available at https://prosecutor-api.mfj.io/


## Authentication

The API is stateless/sessionless. All requests to the system will contain a basic auth element in the request header:

```
Authorization: Basic {base 64 encoded username:password (separated by colon)}
```

So if the username were mel and the password were 12345, the header element would be:

```
Authorization: Basic bWVsOjEyMzQ1
```

Requests without the header or with invalid credentials will receive a 401 response code from the server.

## Endpoints

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

This request sends a file upload part. The content-type of the request is application/octet-stream. The payload contains a portion of the file of the specified size.

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

* minimum part size is 5MB for all but the final part
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

### Upload program flow

Pseudocode for how to upload a file:

```
partSize = 10 MB
partCount = fileSize / partSize
# adjust partSize/partCount if partCount is greater than limit

loop {

    try {

        # start the upload, get which parts have already been uploaded
        startResponse = POST /api/v1/upload/start?id=<id>
        doneParts = startResponse.parts

        # upload any remaining parts
        for ( partNo in [0..partCount] ) {
            if ( partNo not in doneParts ) {
                partStart = partNo * partSize
                partEnd = partStart + partSize # or end of file for the last part

                POST /api/v1/upload/part?id=<id>&partNo=<partNo>&partSize=<partSize>
                # request body is the portion of the file from <partStart> to <partEnd>
            }
        }

        # complete the upload
        loop {
            completeResponse = POST /api/v1/upload/complete
            if ( completeResponse.code == 0 ) {
                break
            } else if ( completeResponse == 2 ) {
                sleep( 30 seconds )
            } else {
                throw nonRecoverableError
            }
        }

        break

    } catch ( recoverableError ) {
        # e.g.: network timeout, 500 from server
        exponentialBackoff()
    }
}
```

## Upload File Structure and Format Example

| Field Name | Data Type | Values |
|---|---|---|
| County | string | County name of prosecutor jurisdiction |
| FileNumber | string | In PBK this is the case number: ###-###### |
| Status | string | e.g. OPEN, CLOSED |
| ReferralDate | date | ISO-8601 date when case was referred to prosecutor office |
| ArrestDate | date | ISO-8601 date of arrest|
| RefAgency | string | Law enforcement agency that referred the case to the prosecutor office |
| Municipality | string | Municipality where the arrest took place |
| AgencyCaseNum | string | Law enforcement or prosecutor office case number |
| Unit | string | Prosecutor office unit dealing witht he case |
| DefendantState | string | Defendant residence state e.g. MO, PA, IL |
| DefendantRace | string | e.g. W, B, I, A, H, U |
| DefendantGender | string | e..g. M, F, U |
| DefendantSID | string | |
| PersonID | integer | Anonymized defendant ID within the county and across data uploads |
| CourtCaseNum | string | Court docket number |
| IncidentDate | date | ISO-8601 date of offense |
| CountNumber | integer | Number of counts |
| LeadChargeFlag | string | Flag for lead or most serious charge e.g., Y, N |
| ReferralCharge | string | Referral charge code |
| ReferralStatute | string | Referral charge statute |
| ReferralChargeDescription | string | Referral charge description |
| ReferralSeverity | string | Referral charge severity e.g. F, M, I, U, L |
| ReferralClass | string | Referral charge class depending on state e.g. A, B, C, D, E, U |
| ReferralModifier | string | Referral charge modifiers/enhancers e.g. Attempted, Conspiracy |
| ReferralNCIC | string | Referral charge NCIC/FBI code. It is numeric with leading zeros |
| ReferralNCICEnhancerDesc | string | Referral charge NCIC/FBI enhancer description |
| ChargeCode | string | Filing charge code |
| ChargeStatute | string | Filing charge statute |
| ChargeDescription | string | Filing charge description |
| Severity | string | Filing charge severity e.g., F, M, I, L |
| Class | string | Filing charge class depending on state e.g., A. B, C, D, U |
| ChargeModifier | string | Filing charge modifiers/enhancers e.g., Attempted, Conspiracy |
| ChargeNCIC | string | Filing charge NCIC/FBI code. It is numeric but uses leading zeros |
| ChargeNCICEnhancerDesc | string | Filing charge NCIC/FBI enhancer description |
| CaseScreeningDecision | string | Prosecutor case screening decision e.g., Accepted, Declined |
| CaseScreeningDate_ReviewOfCharges | date | Prosecutor case screening ISO-8601 date |
| IssuedDate | date | Charge filing ISO-8601 date |
| ChargeDispo | string | Charge disposition |
| DispoDate | date | Charge disposition ISO-8601 date |
| SentenceDate | date | Sentence ISO-8601 date |
| ActConfType | string | Actual confinement type |
| ActConfDays | integer | Actual confinement days |
| ActConfMonths | integer | Actual confinement months |
| ActConfYears | integer | Actual confinement years |
| ActConfStartDate | date | Actual confinement start ISO-8601 date |
| ActProbType | string | Actual probation type |
| ActProbDays | integer | Actual probation days |
| ActProbMonths | integer | Actual probation months |
| ActProbYears | integer | Actual probation years |
| ActProbStartDate | date | Actual probation start ISO-8601 date |
| ActFine | string | Actual fine |
| CaseVicCount | integer | Number of victims in case |
| VictimRace | string | Victim race e.g. W, B, I, A, H, U |
| VictimGender | string | Victim gender e.g. M, F, U |
| AgeAtOffenseDate | integer | Age of defendant at time of offense |
| Domestic | string | Whether case is domestic abuse or not e.g. Yes, No |
| CaseIssuedToDispDays | integer | Number of days from filing to disposition |
| CaseIssuedToSentDays | integer | Number of days from filing to sentence |

Data will be accepted in any of three formats: CSV, JSON or XML.

### CSV

CSV files will use the field names above as headers on the first row, and the file will conform to RFC-4180.

#### CSV Example

All values in this example are blank. Format values per field definition table above.

```csv
County,FileNumber,Status,ReferralDate,ArrestDate,RefAgency,Municipality,AgencyCaseNum,Unit,DefendantState,DefendantRace,DefendantGender,DefendantSID,PersonID,CourtCaseNum,IncidentDate,CountNumber,LeadChargeFlag,ReferralCharge,ReferralStatute,ReferralChargeDescription,ReferralSeverity,ReferralClass,ReferralModifier,ReferralNCIC,ReferralNCICEnhancerDesc,ChargeCode,ChargeStatute,ChargeDescription,Severity,Class,ChargeModifier,ChargeNCIC,ChargeNCICEnhancerDesc,CaseScreeningDecision,CaseScreeningDate_ReviewOfCharges,IssuedDate,ChargeDispo,DispoDate,SentenceDate,ActConfType,ActConfDays,ActConfMonths,ActConfYears,ActConfStartDate,ActProbType,ActProbDays,ActProbMonths,ActProbYears,ActProbStartDate,ActFine,CaseVicCount,VictimRace,VictimGender,AgeAtOffenseDate,Domestic,CaseIssuedToDispDays,CaseIssuedToSentDays
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
```

### JSON

JSON files will be an array of objects, one object per row, using the field names above as keys. The file will conform to RFC-7159.

#### JSON Example

All values in this example are blank. Format values per field definition table above.

```json
[
    {
      "County": "",
      "FileNumber": "",
      "Status": "",
      "ReferralDate": "",
      "ArrestDate": "",
      "RefAgency": "",
      "Municipality": "",
      "AgencyCaseNum": "",
      "Unit": "",
      "DefendantState": "",
      "DefendantRace": "",
      "DefendantGender": "",
      "DefendantSID": "",
      "PersonID": "",
      "CourtCaseNum": "",
      "IncidentDate": "",
      "CountNumber": "",
      "LeadChargeFlag": "",
      "ReferralCharge": "",
      "ReferralStatute": "",
      "ReferralChargeDescription": "",
      "ReferralSeverity": "",
      "ReferralClass": "",
      "ReferralModifier": "",
      "ReferralNCIC": "",
      "ReferralNCICEnhancerDesc": "",
      "ChargeCode": "",
      "ChargeStatute": "",
      "ChargeDescription": "",
      "Severity": "",
      "Class": "",
      "ChargeModifier": "",
      "ChargeNCIC": "",
      "ChargeNCICEnhancerDesc": "",
      "CaseScreeningDecision": "",
      "CaseScreeningDate_ReviewOfCharges": "",
      "IssuedDate": "",
      "ChargeDispo": "",
      "DispoDate": "",
      "SentenceDate": "",
      "ActConfType": "",
      "ActConfDays": "",
      "ActConfMonths": "",
      "ActConfYears": "",
      "ActConfStartDate": "",
      "ActProbType": "",
      "ActProbDays": "",
      "ActProbMonths": "",
      "ActProbYears": "",
      "ActProbStartDate": "",
      "ActFine": "",
      "CaseVicCount": "",
      "VictimRace": "",
      "VictimGender": "",
      "AgeAtOffenseDate": "",
      "Domestic": "",
      "CaseIssuedToDispDays": "",
      "CaseIssuedToSentDays": ""
    },
    {
      "County": "",
      "FileNumber": "",
      "Status": "",
      "ReferralDate": "",
      "ArrestDate": "",
      "RefAgency": "",
      "Municipality": "",
      "AgencyCaseNum": "",
      "Unit": "",
      "DefendantState": "",
      "DefendantRace": "",
      "DefendantGender": "",
      "DefendantSID": "",
      "PersonID": "",
      "CourtCaseNum": "",
      "IncidentDate": "",
      "CountNumber": "",
      "LeadChargeFlag": "",
      "ReferralCharge": "",
      "ReferralStatute": "",
      "ReferralChargeDescription": "",
      "ReferralSeverity": "",
      "ReferralClass": "",
      "ReferralModifier": "",
      "ReferralNCIC": "",
      "ReferralNCICEnhancerDesc": "",
      "ChargeCode": "",
      "ChargeStatute": "",
      "ChargeDescription": "",
      "Severity": "",
      "Class": "",
      "ChargeModifier": "",
      "ChargeNCIC": "",
      "ChargeNCICEnhancerDesc": "",
      "CaseScreeningDecision": "",
      "CaseScreeningDate_ReviewOfCharges": "",
      "IssuedDate": "",
      "ChargeDispo": "",
      "DispoDate": "",
      "SentenceDate": "",
      "ActConfType": "",
      "ActConfDays": "",
      "ActConfMonths": "",
      "ActConfYears": "",
      "ActConfStartDate": "",
      "ActProbType": "",
      "ActProbDays": "",
      "ActProbMonths": "",
      "ActProbYears": "",
      "ActProbStartDate": "",
      "ActFine": "",
      "CaseVicCount": "",
      "VictimRace": "",
      "VictimGender": "",
      "AgeAtOffenseDate": "",
      "Domestic": "",
      "CaseIssuedToDispDays": "",
      "CaseIssuedToSentDays": ""
    }
]
```

### XML

XML files will be a series of Record elements contained within a root Records element. Each Record element will contain a row of data, using the above field names as element names (e.g. <Status>DISPOSED</Status>). The file will conform to the W3C XML 1.0 Specification.

#### XML Example

All values in this example are blank. Format values per field definition table above.

```xml
<Records>
  <Record>
    <County></County>
    <FileNumber></FileNumber>
    <Status></Status>
    <ReferralDate></ReferralDate>
    <ArrestDate></ArrestDate>
    <RefAgency></RefAgency>
    <Municipality></Municipality>
    <AgencyCaseNum></AgencyCaseNum>
    <Unit></Unit>
    <DefendantState></DefendantState>
    <DefendantRace></DefendantRace>
    <DefendantGender></DefendantGender>
    <DefendantSID></DefendantSID>
    <PersonID></PersonID>
    <CourtCaseNum></CourtCaseNum>
    <IncidentDate></IncidentDate>
    <CountNumber></CountNumber>
    <LeadChargeFlag></LeadChargeFlag>
    <ReferralCharge></ReferralCharge>
    <ReferralStatute></ReferralStatute>
    <ReferralChargeDescription></ReferralChargeDescription>
    <ReferralSeverity></ReferralSeverity>
    <ReferralClass></ReferralClass>
    <ReferralModifier></ReferralModifier>
    <ReferralNCIC></ReferralNCIC>
    <ReferralNCICEnhancerDesc></ReferralNCICEnhancerDesc>
    <ChargeCode></ChargeCode>
    <ChargeStatute></ChargeStatute>
    <ChargeDescription></ChargeDescription>
    <Severity></Severity>
    <Class></Class>
    <ChargeModifier></ChargeModifier>
    <ChargeNCIC></ChargeNCIC>
    <ChargeNCICEnhancerDesc></ChargeNCICEnhancerDesc>
    <CaseScreeningDecision></CaseScreeningDecision>
    <CaseScreeningDate_ReviewOfCharges></CaseScreeningDate_ReviewOfCharges>
    <IssuedDate></IssuedDate>
    <ChargeDispo></ChargeDispo>
    <DispoDate></DispoDate>
    <SentenceDate></SentenceDate>
    <ActConfType></ActConfType>
    <ActConfDays></ActConfDays>
    <ActConfMonths></ActConfMonths>
    <ActConfYears></ActConfYears>
    <ActConfStartDate></ActConfStartDate>
    <ActProbType></ActProbType>
    <ActProbDays></ActProbDays>
    <ActProbMonths></ActProbMonths>
    <ActProbYears></ActProbYears>
    <ActProbStartDate></ActProbStartDate>
    <ActFine></ActFine>
    <CaseVicCount></CaseVicCount>
    <VictimRace></VictimRace>
    <VictimGender></VictimGender>
    <AgeAtOffenseDate></AgeAtOffenseDate>
    <Domestic></Domestic>
    <CaseIssuedToDispDays></CaseIssuedToDispDays>
    <CaseIssuedToSentDays></CaseIssuedToSentDays>
  </Record>
  <Record>
    <County></County>
    <FileNumber></FileNumber>
    <Status></Status>
    <ReferralDate></ReferralDate>
    <ArrestDate></ArrestDate>
    <RefAgency></RefAgency>
    <Municipality></Municipality>
    <AgencyCaseNum></AgencyCaseNum>
    <Unit></Unit>
    <DefendantState></DefendantState>
    <DefendantRace></DefendantRace>
    <DefendantGender></DefendantGender>
    <DefendantSID></DefendantSID>
    <PersonID></PersonID>
    <CourtCaseNum></CourtCaseNum>
    <IncidentDate></IncidentDate>
    <CountNumber></CountNumber>
    <LeadChargeFlag></LeadChargeFlag>
    <ReferralCharge></ReferralCharge>
    <ReferralStatute></ReferralStatute>
    <ReferralChargeDescription></ReferralChargeDescription>
    <ReferralSeverity></ReferralSeverity>
    <ReferralClass></ReferralClass>
    <ReferralModifier></ReferralModifier>
    <ReferralNCIC></ReferralNCIC>
    <ReferralNCICEnhancerDesc></ReferralNCICEnhancerDesc>
    <ChargeCode></ChargeCode>
    <ChargeStatute></ChargeStatute>
    <ChargeDescription></ChargeDescription>
    <Severity></Severity>
    <Class></Class>
    <ChargeModifier></ChargeModifier>
    <ChargeNCIC></ChargeNCIC>
    <ChargeNCICEnhancerDesc></ChargeNCICEnhancerDesc>
    <CaseScreeningDecision></CaseScreeningDecision>
    <CaseScreeningDate_ReviewOfCharges></CaseScreeningDate_ReviewOfCharges>
    <IssuedDate></IssuedDate>
    <ChargeDispo></ChargeDispo>
    <DispoDate></DispoDate>
    <SentenceDate></SentenceDate>
    <ActConfType></ActConfType>
    <ActConfDays></ActConfDays>
    <ActConfMonths></ActConfMonths>
    <ActConfYears></ActConfYears>
    <ActConfStartDate></ActConfStartDate>
    <ActProbType></ActProbType>
    <ActProbDays></ActProbDays>
    <ActProbMonths></ActProbMonths>
    <ActProbYears></ActProbYears>
    <ActProbStartDate></ActProbStartDate>
    <ActFine></ActFine>
    <CaseVicCount></CaseVicCount>
    <VictimRace></VictimRace>
    <VictimGender></VictimGender>
    <AgeAtOffenseDate></AgeAtOffenseDate>
    <Domestic></Domestic>
    <CaseIssuedToDispDays></CaseIssuedToDispDays>
    <CaseIssuedToSentDays></CaseIssuedToSentDays>
  </Record>
</Records>
```
