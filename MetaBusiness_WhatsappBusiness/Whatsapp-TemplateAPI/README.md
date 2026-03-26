# Using a Local Node.js Script for Meta Resumable Uploads and Template Creation

This guide explains how to use the Node.js script from the `turivishal/meta-resumable-upload-api` repository to simplify uploading media to Meta (for WhatsApp) and subsequently creating a message template using the uploaded media handle.

This script acts as a local server that handles the two-step resumable upload process required by Meta and provides simple endpoints for uploading and template creation.

## Prerequisites

*   **Node.js and npm:** Ensure you have Node.js (which includes npm) installed. [Download Node.js](https://nodejs.org/)
*   **Git:** Ensure you have Git installed. [Download Git](https://git-scm.com/)

## Setup Instructions

1.  **Clone the Repository:**
    Open your terminal or command prompt and clone the repository:
    ```bash
    git clone https://github.com/turivishal/meta-resumable-upload-api.git
    ```

2.  **Navigate to Directory:**
    Change into the newly cloned directory:
    ```bash
    cd meta-resumable-upload-api
    ```

3.  **Install Dependencies:**
    Install the required Node.js packages:
    ```bash
    npm install
    ```

4.  **Configure Credentials (.env file):**
    *   There should be a file named `.env.example`. Rename it or create a new file named `.env` in the `meta-resumable-upload-api` directory.
    *   Open the `.env` file with a text editor.
    *   Update the file with **your specific Meta credentials**. Replace the placeholder values.

    ```dotenv
    PORT=2002 # You can change this if needed

    # Meta Graph API endpoint and version (Adjust version if necessary)
    META_API_URI=https://graph.facebook.com/v22.0

    # Replace xxxxxxx with YOUR Meta Developer App ID
    META_APP_ID=xxxxxxx

    # Replace xxxxxxx with YOUR WhatsApp Business System User Access Token
    META_ACCESS_TOKEN=xxxxxxx

    # Replace xxxxxxx with YOUR WhatsApp Business Account (WABA) ID
    META_BUSINESS_ACC_ID=xxxxxxx
    ```
    *   Save the `.env` file.

## Usage Instructions

1.  **Run the Local Server:**
    Start the Node.js server from your terminal (make sure you are still in the `meta-resumable-upload-api` directory):
    ```bash
    node index.js
    ```
    You should see a confirmation like `Server listening on port 2002`. **Keep this terminal window open and the server running.**

2.  **Upload Media File:**
    *   Open a **new** terminal window or use a tool like Postman.
    *   Send a `POST` request to the local server's `/uploadMedia` endpoint, attaching your media file as form data.
    *   **IMPORTANT: Replace `/path/to/your/document.pdf`** with the actual, full path to the file you want to upload.

    **Example using `curl`:**
    ```bash
    # Replace the path with YOUR file's path
    curl --location "http://localhost:2002/uploadMedia" --form "file=@\"/path/to/your/document.pdf\""
    ```

    *   **Successful Response:** If the upload via the script is successful, you will receive a JSON response like this:
        ```json
        {
            "message": "Uploaded!",
            "body": {
                "h": "YOUR_UNIQUE_FILE_HANDLE_WILL_BE_HERE"
            }
        }
        ```
    *   **IMPORTANT:** Copy the value of the `"h"` field. This is your **Media Asset Handle**. You need it for the next step.

3.  **Create Message Template:**
    *   Use your preferred API tool (like Postman, Insomnia) or an n8n HTTP Request node.
    *   Send a `POST` request to the local server's endpoint: `http://localhost:2002/createTemplate`.
    *   Set the request header `Content-Type` to `application/json`.
    *   Set the request **Body** to the following **JSON structure**.
    *   **IMPORTANT: Replace `"PASTE_YOUR_FILE_HANDLE_HERE"`** in the `header_handle` array with the actual **Media Asset Handle** you copied in step 2.
    *   **IMPORTANT: Adapt the template `name`, `language`, `category`, `text`, and `example` values to match the template YOU want to create.**

    ---
    **JSON Request Body:**

    Use this structure directly as your JSON payload in your API tool or n8n node.

    ```json
    {
      "name": "your_document_template_name",
      "language": "en_US",
      "category": "UTILITY",
      "components": [
        {
          "type": "HEADER",
          "format": "DOCUMENT",
          "example": {
            "header_handle": [
              "PASTE_YOUR_FILE_HANDLE_HERE" // something like: '4:SkJQIEN....'
            ]
          }
        },
        {
          "type": "BODY",
          "text": "Here is the document for {{1}}.",
          "example": {
            "body_text": [
              ["Customer Name Example"]
            ]
          }
        },
        {
          "type": "FOOTER",
          "text": "Your Company Text"
        }
      ]
    }
    ```
    ---

    *   **Successful Response:** If the template creation is accepted by Meta (via the local script), you will receive a JSON response like this:
        ```json
        {
            "message": "Template Created!",
            "body": {
                "id": "NEW_TEMPLATE_ID",
                "status": "PENDING", // Or APPROVED, REJECTED
                "category": "UTILITY" // The category you specified
            }
        }
        ```

You have now uploaded the media and created a template using the local helper script. You can check the template status in the WhatsApp Business Manager or via the API.


**Need a more advanced deployment, integration, or enterprise automation?** Visit [Stardawnai.com](https://stardawnai.com) for professional consulting and development on AI-driven process automation, SAP integration, self-hosted enterprise N8N workflows, and custom hybrid infrastructure solutions tailored to your business needs.