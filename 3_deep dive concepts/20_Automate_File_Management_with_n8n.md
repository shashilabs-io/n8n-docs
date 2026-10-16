# Chapter 20: Automate File Management with n8n
## Master Binary Data Handling

## Learning objectives

By the end of this chapter, you should be able to:

- Explain binary data in n8n and distinguish it from JSON data.
- Download a file from a public URL using the HTTP Request node.
- Download email attachments with the Gmail node.
- Split multiple binary attachments into separate workflow items.
- Compress files into an archive and decompress an archive.
- Upload files to Google Drive.
- Recognize other file-related nodes and understand when local-disk access is possible.

---

## 1. What is binary data in n8n?

**Binary data** represents file content handled by a workflow. Examples include:

- PDF documents
- PNG and JPG images
- Videos
- ZIP archives
- CSV files

n8n presents file content in the **Binary** output view. Depending on the file type, the editor may let you preview or download it. The binary output also includes metadata such as the file name, extension, MIME type, and size.

### JSON data vs. binary data

| Output | Usually contains | Example |
|---|---|---|
| JSON | Structured text and values | Email subject, sender, message body |
| Binary | File content and file metadata | A PDF attachment or downloaded image |

A workflow item can contain both JSON and binary data. For example, a Gmail result can contain the email's subject and sender in JSON while its attachments are held in binary properties.

### Conceptual flow

```text
Source (URL, email, archive, etc.)
             |
             v
        n8n node
             |
       +-----+------+
       |            |
       v            v
    JSON data    Binary data
 (text/fields)  (file content)
       |            |
       +-----+------+
             |
             v
     Process, compress,
     split, or upload
```

---

## 2. Download a file with the HTTP Request node

Use this pattern when a file is available through a URL that your n8n instance can access.

### Basic setup

1. Add a **Manual Trigger**.
2. Add an **HTTP Request** node after it.
3. Set **Method** to `GET`.
4. Enter the file's URL.
5. Leave authentication and other request fields unset when the resource is genuinely public and does not require them.
6. Execute the node and inspect its output.

When the response is configured/recognized as a file, the content appears in the **Binary** view. The binary property may be named `data`; check the actual output rather than assuming the name. Metadata can include the file name, extension, MIME type, and size.

### Why download files this way?

After the file is available as binary data, later nodes can process it. For example, an image-processing workflow might download images, apply a suitable image-processing step, and upload optimized results to storage.

**Important:** A public URL is not the same as a guaranteed successful download. The URL must be reachable by the n8n environment and return the expected content. Some URLs require authentication, headers, or redirects.

---

## 3. Download Gmail attachments

A Gmail message can contain ordinary message data in JSON and attachments as binary data. Downloading a message does not necessarily mean its attachments have been downloaded as binary output.

### Configure the Gmail node

1. Add a **Gmail** node and select the operation for retrieving messages.
2. Configure the account credentials and choose the message(s) to retrieve.
3. In the node options, turn **Simplify** off if required to expose the relevant attachment options.
4. Add or enable **Download Attachments**.
5. Execute the node and inspect the Binary output.

The exact option labels and layout can vary with n8n versions. Verify the node's output after running it.

### One email, multiple attachments

Suppose one email has nine attachments. The Gmail node can return **one workflow item** for the email, with multiple binary properties holding the nine files. That is different from returning nine workflow items.

```text
Gmail node
   |
   v
1 email item
  JSON: subject, sender, body, ...
  Binary: attachment_0, attachment_1, ... attachment_8
```

The binary property names depend on the node's settings and output. An attachment prefix can be used where supported to control the property-name prefix.

---

## 4. Split attachments into separate items

If you want to process each attachment independently, use **Split Out** with the binary data field as the field to split, as demonstrated in the source workflow.

Conceptually:

```text
Gmail: 1 email item with 9 binary attachments
                    |
                    v
              Split Out
                    |
                    v
             9 file items
```

After splitting, each item can be routed or processed independently. Possible next steps include:

- Use a **Switch** or **IF** node to route files by type.
- Process files in a loop or batch when needed.
- Upload each file to cloud storage.
- Apply different handling to PDFs, images, and other file types.

**Check the output carefully:** Binary properties are not the same thing as a JSON array. Configure Split Out against the actual structure shown in your node output and verify that the result is one item per file. Node behavior and supported fields may differ by n8n version.

The editor may preview some image formats but not PDFs. A missing preview button does not necessarily mean the file was not downloaded; try downloading the file to inspect it.

---

## 5. Compress multiple files

The **Compression** node can compress or decompress binary files. For compression, the example uses ZIP output.

### Typical configuration

1. Add the Compression node after the node that provides the source files.
2. Choose the **Compress** operation.
3. Specify the binary property name(s) that contain the files to include.
4. If the node expects several property names, provide them in the required comma-separated format.
5. Choose the output format, such as `zip` where available.
6. Set an archive file name, such as `gmail-attachments.zip`.
7. Set the output binary property name, such as `data`.
8. Execute and inspect the output's Binary view.

The result is an archive stored in a binary property. The output also has metadata such as file name, MIME type, and size.

### Key detail: property names matter

A binary property name is the key that holds the file. For example, an item might contain properties named `attachment_0`, `attachment_1`, and `attachment_2`. A compression node may expect those names, rather than the literal contents of the files.

If you need to build a dynamic list of binary property names, use a suitable expression or a **Code** node to construct the list in the exact format the Compression node expects. Verify the required input format in your installed node version.

---

## 6. Decompress an archive

To extract the contents of an archive:

1. Add the Compression node.
2. Select the **Decompress** operation.
3. Set the input binary property to the property containing the ZIP file (often `data`, depending on the preceding node).
4. Configure the output prefix if the node offers one—for example, `file_`.
5. Execute and inspect the Binary output.

The decompressed files may be stored as multiple binary properties on a single workflow item. If downstream processing requires one item per extracted file, use **Split Out** with the actual binary structure and confirm the output.

```text
ZIP archive (one binary property)
             |
             v
       Compression:
        Decompress
             |
             v
One item containing multiple extracted binary properties
             |
             v
          Split Out
             |
             v
      One item per file
```

---

## 7. Upload files to Google Drive

A Google Drive node can upload binary files to a chosen Drive folder.

### Basic upload configuration

1. Add the **Google Drive** node.
2. Select the **Upload** operation.
3. Connect valid Google Drive credentials.
4. Set the **Input Data Field Name** to the binary property containing the file—for example, `data` or `file_0`.
5. Set the destination folder.
6. Optionally specify the uploaded file's name.
7. Execute the node and confirm the file appears in the intended folder.

### Uploading multiple files

When a node receives multiple workflow items, it commonly executes once per input item. If every item uses a different binary property name, a fixed field name may work for one item but fail for another.

For example:

```text
Item 1: binary property file_0
Item 2: binary property file_1
Item 3: binary property file_2
```

A Google Drive node configured with the fixed property name `file_0` will not find that property on items 2 and 3. Configure the input field name dynamically when appropriate, using the actual binary keys available on each item. Test with more than one item before enabling the workflow.

The **Execute Once** setting changes execution behavior; it is not a substitute for correct per-item field mapping when the goal is to upload every file. Use it only when running once is truly the intended behavior.

---

## 8. Other file-related nodes

The following node types can support broader file-management workflows. Availability and exact options can vary by n8n version and hosting setup.

| Node / capability | Typical purpose |
|---|---|
| **Read/Write Files from Disk** | Read files from or write files to the filesystem accessible to the n8n process. |
| **Convert to File** | Turn structured JSON data into a binary file, such as a file containing exported data. |
| **Extract From File** | Parse supported file formats and turn their contents into structured data. |
| **Compression** | Compress or decompress files. |
| **FTP / SFTP** | Perform supported file operations with an FTP/SFTP server, such as listing, downloading, uploading, moving, or renaming files. |
| **Google Drive / OneDrive / other storage integrations** | Upload, download, or organize files through the relevant service integration. |

### Local filesystem access and n8n hosting

A cloud-hosted n8n instance does **not** automatically have access to files in your personal computer's Downloads folder. A disk node works with the filesystem available to the n8n process, subject to deployment configuration and permissions.

If you self-host n8n, files may be accessible on the server or in mounted directories, but access is constrained by container mounts, permissions, and security configuration. Do not assume that a path on your own computer is visible inside a remote or containerized n8n instance.

---

## 9. Example workflow patterns

### Pattern A: Download and archive files

```text
Manual Trigger
      |
      v
HTTP Request (download file)
      |
      v
Compression (if the input format and node settings fit)
      |
      v
Cloud storage upload
```

### Pattern B: Save email attachments to cloud storage

```text
Gmail (retrieve messages + download attachments)
      |
      v
Split Out (one item per attachment, if needed)
      |
      v
Optional routing by file type
      |
      v
Google Drive / another storage integration
```

### Pattern C: Extract and process a ZIP archive

```text
HTTP Request or storage node (get ZIP)
                  |
                  v
         Compression: Decompress
                  |
                  v
        Split Out (if needed)
                  |
                  v
       Process or upload each file
```

These are design patterns, not ready-to-import workflows. Configure credentials, node fields, binary property names, and error handling for your own data.

---

## 10. Troubleshooting checklist

### I cannot see a Binary tab

- Confirm the node actually returned file content.
- Check the HTTP Request response configuration and the URL's response.
- For Gmail, verify that attachment downloading is enabled.
- Execute the node again and inspect its latest output.

### The email has attachments, but only one item appears

That may be expected: one email is one workflow item, and its attachments can be stored as multiple binary properties. Split the binary data into separate items if the next step requires one item per attachment.

### A PDF has no preview button

The editor may not preview that format. Download it and open it with a suitable application.

### Compression cannot find a file

- Check the exact binary property names.
- Confirm that the Compression node expects the format you supplied.
- If the node expects a comma-separated list, do not assume that writing `$binary` automatically generates the list.
- Inspect the incoming item's Binary output before changing expressions.

### Upload fails for the second file

- Check whether each item has the binary property name configured in the upload node.
- Use a dynamic property-name expression if the key varies between items.
- Test the workflow with several items.
- Use **Execute Once** only if a single execution is actually intended.

### A disk node cannot find a file on my computer

The n8n process must be able to access that path. A cloud-hosted instance cannot automatically read your local computer's filesystem.

---

## 11. Practice questions

1. What is binary data in n8n? Give three examples.
2. What is the difference between JSON output and Binary output?
3. Why can one Gmail item contain several attachments?
4. When would you use Split Out after downloading attachments?
5. What information does file metadata commonly include?
6. What must you check when a Compression node cannot find a file?
7. Why might a fixed binary field name fail when uploading several files?
8. Does a cloud-hosted n8n instance automatically access your computer's local files?

### Answer key

1. Binary data represents file content handled by n8n, such as PDFs, images, or ZIP archives.
2. JSON holds structured text and values; Binary holds file content and associated file metadata.
3. The workflow item represents the email, while attachments are stored as multiple binary properties on that item.
4. Use it when downstream steps need to handle each attachment as a separate workflow item.
5. Common examples include file name, extension, MIME type, and file size.
6. Check the actual binary property names and the input format expected by the node.
7. Different items may contain different binary keys, so the configured key may be absent from some items.
8. No. The n8n process must have access to the filesystem; cloud hosting does not automatically expose the user's local disk.

---

## Chapter summary

- Binary data is how n8n carries file content such as images, PDFs, and ZIP archives.
- The Binary view lets you inspect file content and metadata.
- HTTP Request can fetch a file from an accessible URL.
- Gmail can download message attachments as binary properties.
- One email item can contain many attachments; split them when separate items are needed.
- Compression can create archives, and decompression can extract their contents.
- Google Drive and other integrations can upload or organize files when their binary field mapping is correct.
- Local disk access depends on where n8n runs and which paths the process can access.
