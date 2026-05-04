<img width="1691" height="625" alt="image" src="https://github.com/user-attachments/assets/ed196396-c208-43cc-bcc2-98919309654a" />
AI-Powered Insurance Lead Generation & Follow-Up System: n8n Workflow Design

This document outlines the n8n workflow for the AI-Powered Insurance Lead Generation & Follow-Up System, incorporating the user's requirements for lead capture, AI-driven personalized email generation, human review and editing, and automated delivery.

1. Goal of the Automation

The primary goal of this automation is to efficiently capture insurance leads from a landing page, generate personalized email responses using AI, allow for human review and editing of these emails, and then send them to the leads via SendGrid, while tracking the entire process in Google Sheets.

2. Workflow Overview

Upon a new lead submission from the Laravel landing page, an n8n workflow is triggered. This workflow first stores the lead's details in Google Sheets. It then uses the Gemini API to generate a personalized email draft. This draft is presented to the business owner via a web form for review and editing. Once approved and potentially edited, the email is sent to the lead using SendGrid, and the Google Sheet is updated with the email content, approval status, and sending status.

3. n8n Node Architecture

The n8n workflow will consist of the following nodes, executed sequentially:

1.Webhook (Trigger Node): Receives lead data from the Laravel landing page.

2.Google Sheets (Append Row): Stores the initial lead data.

3.Gemini (Generate Content): Generates a personalized email draft based on lead data and product information.

4.Wait (for Webhook Call): Pauses the workflow and generates a unique URL for human approval and editing.

5.Webhook (Response): Sends the AI-generated email draft to the owner for review via a custom form.

6.Google Sheets (Update Row): Updates the lead record with the approval status and final email content.

7.IF (Conditional Logic): Checks if the email was approved.

8.SendGrid (Send Email): Sends the approved email to the lead.

9.Google Sheets (Update Row): Updates the lead record with the email sending status.




4. Step-by-Step Implementation

    Here’s how to configure each node and connect them:
    
    Node 1: Webhook (Trigger Node)
    
    •Purpose: To receive lead data from the Laravel landing page.
    
    •Configuration:
    
    •Webhook URL: n8n will provide a unique URL. Configure your Laravel backend to send a POST request to this URL with the lead data (Name, Email, Phone number, Insurance interest) in JSON format.
    
    •HTTP Method: POST
    
    •Authentication: None (or Basic Auth if preferred for added security, but ensure Laravel sends credentials).
    
    •Output: The lead data (e.g., {{ $json.name }}, {{ $json.email }}, {{ $json.phone }}, {{ $json.insuranceInterest }}).

Node 2: Google Sheets (Append Row)

    •Purpose: To store the initial lead data in a Google Sheet.
    
    •Configuration:
    
    •Operation: Append Row
    
    •Authentication: OAuth2 (connect your Google account).
    
    •Spreadsheet ID: Select your target Google Sheet.
    
    •Sheet Name: Specify the sheet where leads are stored (e.g., "Leads").
    
    •Values: Map the incoming webhook data to your Google Sheet columns. Ensure you also add columns for Email Draft, Approval Status, Approved Email Content, and Sent Status.
    
    •Name: {{ $json.name }}
    
    •Email: {{ $json.email }}
    
    •Phone Number: {{ $json.phoneNumber }}
    
    •Insurance Interest: {{ $json.insuranceInterest }}
    
    •Email Draft: (Leave empty for now)
    
    •Approval Status: "Pending"
    
    •Approved Email Content: (Leave empty for now)
    
    •Sent Status: "Not Sent"
    
    •Row ID: {{ $node["Google Sheets"].json.row }} (This is crucial for updating the specific row later).
    
    •Output: The appended row's data, including the Row ID which will be used for subsequent updates.

Node 3: Gemini (Generate Content)

    •Purpose: To generate a personalized email draft.
    
    •Configuration:
    
    •Authentication: Gemini API Key (ensure you have this set up in n8n).
    
    •Model: gemini-pro (or gemini-1.5-pro if available and preferred).
    
    •Prompt: Craft a detailed prompt that includes the lead's information and your insurance product details. Example:
    
    Plain Text
    
    
    You are an insurance agent writing a personalized follow-up email to a new lead. The lead's name is {{ $json.name }}, their email is {{ $json.email }}, and they are interested in {{ $json.insuranceInterest }}. 
    
    Our company offers comprehensive insurance plans including:
    - **Life Insurance**: Covers unexpected events, ensuring financial security for families. Benefits include tax advantages and flexible payment options.
    - **Health Insurance**: Provides coverage for medical expenses, hospital stays, and preventative care. Includes options for dental and vision.
    - **Auto Insurance**: Protects against financial loss in case of an accident or theft. Customizable policies for various vehicle types.
    
    Draft a professional, friendly, and concise email. Address the lead by name, acknowledge their interest, briefly explain how our {{ $json.insuranceInterest }} product can benefit them, and include a clear call to action to schedule a call or learn more. Keep the tone reassuring and informative. Do not include a signature or contact details, as these will be added automatically.
    
    
    •Output: The AI-generated email content.

Node 4: Wait (for Webhook Call)

    •Purpose: To pause the workflow and wait for human input (approval/editing).
    
    •Configuration:
    
    •Mode: Respond to Webhook
    
    •Webhook URL: This node will generate a unique URL. This URL will be used to create the approval form.
    
    •Timeout: Set an appropriate timeout (e.g., 24 hours) after which the workflow will continue without approval if no response is received.   
    
    
    •Output: This node will output the data received from the approval form (approved status, edited email content).

Node 5: Webhook (Response) - Approval Form

    •Purpose: To present the AI-generated email to the business owner for review and editing, and to receive their decision.
    
    •Configuration:
    
    •HTTP Method: GET (for displaying the form) and POST (for submitting the form).
    
    •Path: Use a custom path (e.g., /approve-email).
    
    •Response Mode: Raw
    
    •Response Body: This is where you'll create a simple HTML form. The form should include:
    
    HTML
    
    
    <!DOCTYPE html>
    <html>
    <head>
        <title>Review Email</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; background-color: #f4f4f4; }
            .container { background-color: #fff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); max-width: 800px; margin: auto; }
            textarea { width: 100%; height: 300px; margin-bottom: 10px; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
            input[type="submit"] { background-color: #4CAF50; color: white; padding: 10px 15px; border: none; border-radius: 4px; cursor: pointer; font-size: 16px; }
            input[type="submit"]:hover { background-color: #45a049; }
            .reject-button { background-color: #f44336; margin-left: 10px; }
            .reject-button:hover { background-color: #da190b; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Review Lead Email</h1>
            <form action="{{ $node["Wait"].json.webhookUrl }}" method="POST">
                <input type="hidden" name="leadId" value="{{ $node["Google Sheets"].json.row }}">
                <label for="emailContent">Edit Email Content:</label>  
    
                <textarea id="emailContent" name="emailContent">{{ $node["Gemini"].json.candidates[0].content.parts[0].text }}</textarea>  
    
                
                <input type="radio" id="approve" name="status" value="approved" checked>
                <label for="approve">Approve</label>  
    
                <input type="radio" id="reject" name="status" value="rejected">
                <label for="reject">Reject</label>  
      
    
    
                <input type="submit" value="Submit Approval">
            </form>
        </div>
    </body>
    </html>
    
    
    
    •A hidden field for the webhookUrl from the Wait node (Node 4) to send the response back.
    
    •A text area pre-filled with the AI-generated email ({{ $node["Gemini"].json.candidates[0].content.parts[0].text }}).
    
    •A checkbox or radio buttons for "Approve" / "Reject".
    
    •A submit button.
    
    •Important: You will need to construct the URL for this form and send it to the business owner (e.g., via email or a messaging app). The URL will combine your n8n instance URL, the custom path, and query parameters for the webhookUrl from the Wait node and the rowId from the Google Sheets node.





Node 6: Google Sheets (Update Row)

    •Purpose: To update the lead record with the approval status and the final (potentially edited) email content.
    
    •Configuration:
    
    •Operation: Update Row
    
    •Authentication: OAuth2 (same as before).
    
    •Spreadsheet ID: Select your target Google Sheet.
    
    •Sheet Name: "Leads".
    
    •Row ID: {{ $node["Google Sheets"].json.row }} (from the initial append operation).
    
    •Values: Map the data received from the approval form.
    
    •Aproval Status: {{ $json.status }} (from the form submission)
    
    •Approved Email Content: {{ $json.emailContent }} (from the form submission)
    
    •Output: The updated row data.

Node 7: IF (Conditional Logic)
    
    •Purpose: To check if the email was approved before proceeding to send it.
    
    •Configuration:
    
    •Value 1: {{ $json.status }} (from the approval form submission).
    
    •Operation: Is Equal
    
    •Value 2: approved
    
    •Output: Two branches: one for true (approved) and one for false (rejected).

Node 8: SendGrid (Send Email) - (Connected to IF 'True' branch)

    •Purpose: To send the approved email to the lead.
    
    •Configuration:
    
    •Authentication: SendGrid API Key (set up in n8n).
    
    •From Email: Your business email address.
    
    •From Name: Your business name.
    
    •To Email: {{ $node["Webhook"].json.email }} (from the initial lead data).
    
    •Subject: "Regarding your interest in {{ $node["Webhook"].json.insuranceInterest }} Insurance"
    
    •HTML Body: {{ $node["Google Sheets1"].json.approvedEmailContent }} (from the updated Google Sheet row).
    
    •Personalization: Add a professional signature and contact details here.   
        
    •Output: Confirmation of email sent.

Node 9: Google Sheets (Update Row) - (Connected to SendGrid)

    •Purpose: To update the lead record with the final sending status.
    
    •Configuration:
    
    •Operation: Update Row
    
    •Authentication: OAuth2.
    
    •Spreadsheet ID: Select your target Google Sheet.
    
    •Sheet Name: "Leads".
    
    •Row ID: {{ $node["Google Sheets"].json.row }}.
    
    •Values:
    
    •Sent Status: "Sent"
    
    •Output: The final updated row data.

5. Data Flow Explanation

1.Lead Capture: The Laravel landing page sends a JSON payload containing name, email, phoneNumber, and insuranceInterest to the Webhook trigger node.

2.Initial Storage: This data, along with initial Pending status and empty email fields, is appended to a row in Google Sheets. The Row ID of this new row is crucial and is passed along.

3.AI Generation: The lead's name, email, and insuranceInterest are passed to the Gemini node, which uses a detailed prompt (including product information) to generate an email draft.

4.Human Approval Setup: The workflow pauses at the Wait node, generating a unique webhookUrl. The AI-generated email draft and the Row ID are then used to construct an approval form URL, which is presented to the business owner.

5.Owner Review: The owner accesses the form, reviews the emailContent, makes edits if necessary, selects approved or rejected, and submits the form. This submission hits the webhookUrl generated by the Wait node.

6.Status Update: The data from the approval form (status, emailContent) is used to update the corresponding row in Google Sheets, marking the Approval Status and storing the Approved Email Content.

7.Conditional Sending: An IF node checks the Approval Status. If approved, the workflow proceeds to SendGrid.

8.Email Delivery: SendGrid uses the Approved Email Content and the lead's email to send the personalized email.

9.Final Status Update: After sending, the Google Sheet is updated one last time, setting the Sent Status to "Sent". If rejected, the workflow simply ends after the status update in step 6.

6. Testing the Workflow

To thoroughly test this workflow, follow these steps:

1.Set up n8n: Ensure n8n is running and you have configured credentials for Google Sheets, Gemini, and SendGrid.

2.Create Google Sheet: Prepare a Google Sheet with columns: Name, Email, Phone Number, Insurance Interest, Email Draft, Approval Status, Approved Email Content, Sent Status, Row ID.

3.Activate Webhook: Activate the Webhook trigger node in n8n to get its URL.

4.Simulate Lead Submission: Use a tool like Postman, Insomnia, or a simple script to send a POST request to your n8n Webhook URL with sample lead data. Alternatively, integrate with your Laravel landing page for real submissions.

5.Monitor Google Sheets: Verify that a new row is appended with the initial lead data and "Pending" status.

6.Access Approval Form: The n8n execution log will show the URL for the approval form (from the Webhook Response node). Copy this URL and open it in a browser.

7.Review and Edit: On the form, verify the AI-generated email content. Make some edits, select "Approve", and submit.

8.Monitor Google Sheets (again): Check that the Approval Status and Approved Email Content columns are updated in the corresponding row.

9.Check Email: Verify that the lead (your test email address) receives the email via SendGrid with the approved and edited content.

10.Monitor Google Sheets (final): Confirm that the Sent Status is updated to "Sent".

11.Test Rejection: Repeat steps 4-6, but this time select "Reject" on the approval form. Verify that the email is not sent and the Approval Status is updated to "Rejected" in Google Sheets.

7. Possible Improvements

    •Error Handling: Implement robust error handling for each API call (Gemini, Google Sheets, SendGrid) using n8n's error handling features. This could involve sending notifications to an admin, retrying operations, or logging errors to a separate sheet.
    
    •Dynamic Product Information: Instead of hardcoding product details in the Gemini prompt, consider fetching them dynamically from a database or a configuration file based on the insuranceInterest. This makes the system more flexible and easier to update.
    
    •Templating for Emails: While Gemini generates content, you could use a templating engine (e.g., Handlebars in n8n) to ensure consistent branding, headers, and footers for all emails, injecting the AI-generated body into a predefined template.
    
    •Approval Notification: Instead of manually getting the approval URL from the n8n execution log, automatically send an email or a message to the business owner with the approval form link using an additional Email Send or Messaging node (e.g., Slack, Telegram) after the Wait node.
    
    •Lead Scoring: Integrate a lead scoring mechanism. Based on the insuranceInterest or other lead data, assign a score. High-scoring leads could potentially bypass human approval (with caution) or trigger additional follow-ups.
    
    •CRM Integration: As mentioned in your project description, integrating with a CRM like HubSpot or Zoho would centralize lead management and provide a more comprehensive view of customer interactions.
    
    •User Interface for Approval: For a more polished experience, instead of a raw HTML form, you could build a simple frontend application (e.g., with React/Vue) that interacts with an n8n webhook to display and submit approval requests. This offers better UI/UX.

8. Learning Notes

This workflow demonstrates several key automation concepts:

•Event-Driven Architecture: The workflow is triggered by an external event (lead submission), making it reactive and efficient.

•API Integration: Seamlessly connects different services (Laravel, Google Sheets, Gemini, SendGrid) using their respective APIs.

•Human-in-the-Loop Automation: The Wait node combined with a Webhook Response for an approval form is a powerful pattern for incorporating human oversight and decision-making into automated processes. This ensures quality control and allows for critical adjustments before automated actions are finalized.

•Conditional Logic: The IF node is essential for creating dynamic workflows that adapt based on specific conditions (e.g., approved vs. rejected).

•Data Persistence and Transformation: Data is captured, stored, transformed (AI generation), and updated across different nodes and services, showcasing a complete data lifecycle within the automation.

This design provides a robust foundation for your AI-Powered Insurance Lead Generation & Follow-Up System. Feel free to ask if you have any questions or require further modifications.

