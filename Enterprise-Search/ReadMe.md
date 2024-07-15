# Search & Conversation 
OOB service with document search capabilities. This write up will focus on the using JSONL format which is critical for datastore reading and formatting the API search request to the discovery engine.

## JSONL 
In this senario, assuming we have a bucket of objects that contain meta data.

In order for your documents to be able loaded onto Search & Conversation, objects in the folder will need to be compressed into a single JSONL file as mentioned in their documentation. The format of the jsonl in each line has to follow similar to the format like that on their site : 


https://cloud.google.com/generative-ai-app-builder/docs/agent-data-store

Code Snipplet
```
    data = {
            "id": hash(fileid+filedate),
                "jsonData": json.dumps(metadata),
                "content": {
                    "mimeType": file_type,
                    "uri": gcs_uri
                }
            }

```

The key to uploading this information is first to create blobs of 'text/plain' types with the name attached with the .jsonl extention. Decode each line into a utf-8 format and print each line to the jsonl named file. 

```
blob = bucket.blob(folder + jsonl_file_name)
blob.content_type = 'text/plain'

blob.upload_from_string(b'\n'.join([json.dumps(json_data).encode('utf-8') for json_data in jsonl_data]).decode('utf-8').encode('utf-8'))

```

This file can then be uploaded into the datastore by selecting from GCS. 

## Request reformating using python request library  

The problem with this that there is no documentation about using their API with a rest call. There is also a need to attach a credential variable for authentication calls with a service account scope.

This is a typical authentication code from google.

```
import google.auth
import google.auth.transport.requests
from google.oauth2 import service_account

SCOPES = scope_list
SERVICE_ACCOUNT_FILE = file_path
credentials = service_account.Credentials.from_service_account_file(SERVICE_ACCOUNT_FILE, scopes=SCOPES)

auth_req = google.auth.transport.requests.Request()
credentials.refresh(auth_req)

```

Using the credentials, it needs to be added to the request call. 

An example request call provided is using CURL extracted from google:
https://cloud.google.com/generative-ai-app-builder/docs/preview-search-results#genappbuilder_search-rest

```
curl -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" \
-H "Content-Type: application/json" \
"https://discoveryengine.googleapis.com/v1beta/projects/PROJECT_ID/locations/global/collections/default_collection/dataStores/DATA_STORE_ID/servingConfigs/default_search:search" \
-d '{
"servingConfig": "projects/PROJECT_ID/locations/global/collections/default_collection/dataStores/DATA_STORE_ID/servingConfigs/default_search",
"query": "QUERY",
"pageSize": "PAGE_SIZE",
"offset": "OFFSET",
"orderBy": "ORDER_BY",
"params": {"user_country_code": "USER_COUNTRY_CODE",
"searchType": "SEARCH_TYPE"},
"filter": "FILTER"
}'

```


How to translate the CURL to python request with headers and body :

```

headers = {
            'Authorization': 'Bearer {}'.format(credentials.token),
            'Content-Type': 'application/json'
        }
        request_body = {
            'query': {'input': QUERY},
            'summarySpec': {
                'ignoreAdversarialQuery': True,
                'includeCitations': True
            }
        }
        response = requests.post(
            f"https://discoveryengine.googleapis.com/v1alpha/projects/{PROJECT_ID}/locations/{LOCATION}/collections/default_collection/dataStores/{DS_ID}/{CONVO_ID}/-:converse",
            headers=headers,
            json=request_body
        )

```

The additional parameters can be specified in the summary spec.

You might ask what is the data store for? 
The datastore is used mainly for the purpose of storing structued meta. It also helps with the indexing for retrival to improve retrival. Storing User-Defined metadata, custom fields or tags for categorization.

Note : this feature is still in development phrase. Thus, documentations about this feature will take time to siphon through.
