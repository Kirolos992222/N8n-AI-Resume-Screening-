# N8N Resume Screening Automation — Technical README

> **Purpose**: Automate resume collection from Google Drive subfolders (one subfolder per job title), convert each PDF to text, extract structured fields with an LLM (Gemini), and append results to a Google Sheet.

> **Trigger**: Manual — the workflow runs when you click **Execute Workflow** (or execute a specific node during development).


---

## 1) High‑Level Flow

1. **Manual Trigger** → starts the job on demand.
2. **Google Drive: Search main folder** → finds all **job subfolders** inside the specified main folder.
3. **Loop Over Items** → iterates each subfolder.
4. **Google Drive: Search in subfolder** → lists all CV PDFs in the current subfolder.
5. **Google Drive: Download file** → downloads each CV (binary) to pass downstream.
6. **PDF.co: Upload file** → uploads the PDF to PDF.co and returns a temporary URL.
7. **PDF.co: PDF → Text** → converts the PDF file (by URL) into plain text.
8. **Google Gemini (Basic LLM Chain)** → extracts a **JSON** object from the CV text using an HR‑assistant system prompt.
9. **Code** → parses/normalizes the LLM output and assembles a sheet row.
10. **Google Sheets: Append** → appends the structured row to the target spreadsheet.

The workflow expects the following **folder structure** on Google Drive:

```
/Main Folder 
  /Legal Affairs
    CV1.pdf
    CV2.pdf
  /Human resources
    CV3.pdf
    CV4.pdf
```

---

## 2) Prerequisites

* An **n8n** instance (Cloud or self‑hosted) with access to Credentials.
* A Google Drive **main folder** containing job‑titled subfolders with CV PDFs.
* A **Google Sheet** (target) with columns ready to receive extracted fields.
* Accounts/keys for: **Google (Drive/Sheets)**, **PDF.co**, and **Google AI Studio (Gemini)**.

---

## 3) Credentials Setup (n8n)

Below are exact steps to create and attach credentials in n8n for each service. You can use OAuth (recommended) or a Service Account for Google. Pick **one** method per product.

### 3.1 Google — Drive & Sheets

####  OAuth2 

1. Go to **Google Cloud Console** → Create (or choose) a project.
2. Enable APIs: **Google Drive API** and **Google Sheets API**.
3. Configure **OAuth consent screen** (External ). 
4. Create **Credentials → OAuth client ID → Web application**.
5. Add Authorized redirect URI for n8n (format):

   * `https://<YOUR_N8N_HOST>/rest/oauth2-credential/callback`
   * For n8n Cloud, copy the exact redirect shown in the credential form.
6. In n8n, go to **Credentials → New**:

   * Choose **Google Drive OAuth2 API** (for Drive nodes) and **Google Sheets OAuth2 API** (for Sheets nodes).
   * Paste the **Client ID** and **Client Secret**.
   * Click **Connect OAuth2 Account** and complete the Google consent.

> Tip: You can reuse the same OAuth client across both Drive and Sheets credentials.


### 3.2 PDF.co (API Key)

1. Sign in to **PDF.co** and obtain your **API Key**.
2. In n8n, create a new **PDF.co API** credential and paste the key.

### 3.3 Google Gemini (Google AI Studio API Key)

1. Go to **Google AI Studio** and create an **API key**.
2. In n8n, create a **Google AI Studio** (Gemini) credential and paste the key.

> The workflow uses the **Google Gemini Chat Model** node in n8n’s AI stack.

---

## 4) Node‑by‑Node Setup (exact settings)


###4.1 **When clicking ‘Execute workflow’** (Manual Trigger)

* **Node**: *Manual Trigger* (no configuration).

### 4.2 **Search files and folders** — *Accessing the main folder of CVs*

* **Node**: *Google Drive*
* **Resource**: File
* **Operation**: List
* **Filters**:

  * **Parent**: `={{ $env.MAIN_FOLDER_ID || "<MAIN_FOLDER_ID>" }}`
  * **Only Folders**: ✅
  * **Return All**: ✅
* **Credential**: Google Drive (OAuth2 or Service Account)

**Output**: An array of subfolder items. Each item contains at minimum `id`, `name`.

### 4.3 **Loop Over Items** — *Looping over each subfolder*

* **Node**: *Loop Over Items*
* **Items**: From previous node (subfolders list)
* **Batch Size**: Set to a comfortable number (e.g., 1–5) if you want to throttle processing.

### 4.4 **Search files and folders1** — *Extract all resume from each subfolder*

* **Node**: *Google Drive*
* **Resource**: File
* **Operation**: List
* **Filters**:

  * **Parent**: `={{ $json["id"] }}` *(the current subfolder ID from the Loop)*
  * **Mime Type**: `application/pdf`
  * **Return All**: ✅
* **Add field** (optional): `webViewLink` to capture a link to the file.

> If subfolders contain non‑PDFs, add an **IF** node to keep only `mimeType === 'application/pdf'`.

### 4.5 **Set Node** — *Setting the id of each resume to be used in the next node*

* **Node**: *Set*
* **Keep Only Set**: ❌ (optional)
* **Values to set**:

  * `fileId` → `={{ $json["id"] }}`
  * `fileName` → `={{ $json["name"] }}`
  * `jobTitle` → `={{ $parent.item.json.name }} ` *(the subfolder name from the outer loop)*
  * `fileUrl` → `={{ $json["webViewLink"] || "" }}`

### 4.6 **Download file** — *Downloading the CVs*

* **Node**: *Google Drive*
* **Resource**: File
* **Operation**: Download
* **File ID**: `={{ $json["fileId"] }}`
* **Download**: ✅ *(produces `binary.data`)*

### 4.7 **PDFco Api** — *Uploading the CVs to the free pdf extractor (PDF.co)*

* **Node**: *PDF.co*
* **Operation**: Upload a file
* **Input**: *Binary property* = `data`
* **Output**: returns a URL (e.g., `url` or `presignedUrl`) to the uploaded file
* **Credentials**: PDF.co API Key

### 4.8 **PDFco Api1** — *Converting PDF to text*

* **Node**: *PDF.co*
* **Operation**: PDF to Text
* **Source**: *URL*
* **URL**: `={{ $json["url"] }}` *(output from previous PDF.co node)*
* **Text Output**: Choose *Plain Text* (or JSON lines if preferred)

**Output**: a big text string (e.g., under `body`, `text`, or `data` depending on node version).

### 4.9 **Basic LLM Chain** — *Google Gemini Chat Model*

* **Node**: *Google Gemini Chat Model* (n8n AI)
* **Credential**: Google AI Studio API Key
* **Model**: `={{ $env.GEMINI_MODEL || "gemini-2.5-flash" }}`
* **System Instruction**: *(see prompt below)*
* **User Message**: pass the plain‐text CV. Example:

  * `Input Text`: `={{ $json["text"] || $json["data"] || $json["body"] }}` *(use the correct property name from 5.8)*


> You can add role‑specific criteria inside the system prompt (e.g., must‑have skills) and read the **job title** from the `jobTitle` variable when building the prompt: `={{ "Job Title: " + $json.jobTitle }}`

### 4.10 **Code** — *Extracting the text from the LLM into a JSON object*

* **Node**: *Code* (JavaScript)
* **Purpose**: Parse the LLM output (string JSON), normalize types, and produce a single row for Google Sheets.

### 4.11 **Append row in sheet** — *Appending the extract object to Google Sheets*

* **Node**: *Google Sheets*
* **Operation**: *Append*
* **Spreadsheet ID**: `={{ $env.SHEET_ID || "<SHEET_ID>" }}`
* **Sheet**: `={{ $env.SHEET_TAB || "Screening" }}`
* **Value Range / Input Data**: Use the **Code** node output mapping as one row.
* **Credentials**: Google Sheets credential created in §3.1.

> Ensure the Sheet has matching columns:
> `full_name	email_address	phone_number	social_profiles	location	nationality	domain	education	years_of_experience	current_company	past_companies	job_titles	skills	languages_known	career_summary							`

---

