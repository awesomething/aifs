# Roadmap

- [ ] Add file source to chunk metadata, including page number if relevant, and ensure this is displayed when running `search`.
- [ ] Handle file deletions (I think we can just more robustly detect indexed files vs present files)
- [ ] Add tqdm to each step
- [ ] Allow you to ignore certain files or paths (I think path should just accept a list, and if it's a list of files, those are the only files it will take. Then you can write your own logic for ignoring stuff)
- [ ] Make it work with just one file, like pointing to a single PDF.
- [ ] Put an `_.aifs` in each subdirectory, so indexing a path will index all subpaths
- [ ] Support multimodal — transcribe videos and audio, describing each scene / sound, describe images

If you want to upload all content from the PDF, including text outside of tables, you can modify the code to extract and aggregate all the content into a single JSON object before uploading it to Azure Storage. Here's how you can adjust the code:

```python
from azure.core.credentials import AzureKeyCredential
from azure.ai.formrecognizer import DocumentAnalysisClient
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient
import os
import json

# Azure Form Recognizer credentials
form_recognizer_endpoint = os.environ["AZURE_FORM_RECOGNIZER_ENDPOINT"]
form_recognizer_key = os.environ["AZURE_FORM_RECOGNIZER_KEY"]

# Azure Storage credentials
storage_connection_string = os.environ["AZURE_STORAGE_CONNECTION_STRING"]
container_name = "your-container-name"

# Initialize DocumentAnalysisClient
document_analysis_client = DocumentAnalysisClient(
    endpoint=form_recognizer_endpoint, credential=AzureKeyCredential(form_recognizer_key)
)

# Initialize BlobServiceClient and ContainerClient
blob_service_client = BlobServiceClient.from_connection_string(storage_connection_string)
container_client = blob_service_client.get_container_client(container_name)

# Path to the sample documents
path_to_sample_documents = "path/to/sample/documents"

# Analyze documents
with open(path_to_sample_documents, "rb") as f:
    poller = document_analysis_client.begin_analyze_document("prebuilt-layout", document=f)
result = poller.result()

# Initialize an empty list to store extracted content
all_content = []

# Extract content from the document
for page in result.pages:
    page_content = {
        "page_number": page.page_number,
        "width": page.width,
        "height": page.height,
        "content": []  # List to store content on this page
    }
    for line in page.lines:
        page_content["content"].append({
            "type": "line",
            "text": line.content,
            "bounding_box": line.bounding_box,
            "confidence": line.confidence
        })
    for selection_mark in page.selection_marks:
        page_content["content"].append({
            "type": "selection_mark",
            "state": selection_mark.state,
            "bounding_box": selection_mark.bounding_box,
            "confidence": selection_mark.confidence
        })
    # Add content from other elements like tables, if available
    # Add logic here to extract content from other elements like tables, if available
    all_content.append(page_content)

# Convert all content to JSON string
json_data = json.dumps(all_content)

# Upload all content JSON data to Azure Storage
blob_name = "all_content.json"
blob_client = container_client.get_blob_client(blob_name)
blob_client.upload_blob(json_data, overwrite=True)

# Print the uploaded JSON data (optional)
print(json_data)
```

In this code:

- After analyzing the document, we iterate over each page and extract the content.
- For each page, we create a dictionary `page_content` to store information about the page and its content.
- Inside the loop, we extract content from lines and selection marks. You can add additional logic here to extract content from other elements like tables, if available.
- All extracted content is stored in the `all_content` list.
- Finally, we convert the `all_content` list to a JSON string and upload it as a single blob to Azure Storage.