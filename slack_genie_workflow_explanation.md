# Slack Genie Integration Workflow

This n8n workflow connects Slack with Databricks Genie, allowing users to ask questions in Slack and get AI-powered responses based on their data.

## Overview

The workflow listens for Slack app mentions, sends questions to Databricks Genie, polls for results, and sends formatted responses back to Slack threads. A separate error workflow ensures that any failures are gracefully communicated back to users in Slack.

## Workflow Architecture

The system consists of two interconnected workflows:

### Main Workflow: Slack Genie Integration
Handles the full lifecycle of a user question:
1. Receives Slack mentions
2. Routes to new or existing Genie conversations
3. Polls for responses with exponential backoff
4. Processes and formats results (text or query data)
5. Sends formatted responses back to Slack

### Error Workflow: Slack Genie Error Handler
Provides graceful failure handling:
1. Catches any errors from the main workflow
2. Retrieves execution context
3. Notifies users in the original Slack thread

Both workflows work together to ensure users always receive feedback, whether successful or not.

---

## Workflow Sections

### 1. **Trigger & Setup** (Slack Message Reception)

#### **Slack Socket Mode Trigger**
- Listens for `app_mention` events in Slack
- Triggers when someone mentions the bot in a Slack channel

#### **Get a channel**
- Retrieves channel information from Slack
- Gets channel ID and name for the message

#### **Set Slack Event**
- Extracts and stores key information from the Slack event:
  - `channel_id` - Which channel the message came from
  - `event_ts` - Timestamp of the message
  - `thread_ts` - Thread timestamp (if it's a reply in a thread)
  - `channel_name` - Human-readable channel name

#### **Set Genie Space**
- Sets the Databricks Genie Space ID to use
- Configure this with your specific Genie Space ID

---

### 2. **Conversation Management** (Thread Tracking)

#### **Set Backoff Params**
- Initializes polling/retry parameters:
  - `retryCount`: 0 (how many times we've checked)
  - `initialDelay`: 5 seconds (first wait time)
  - `maxDelay`: 60 seconds (maximum wait time)
  - `totalElapsed`: 0 seconds (total time spent polling)
  - `maxTotal`: 600 seconds (10 minute timeout)

#### **Select Existing Conversations**
- Queries PostgreSQL database to check if this Slack thread already has a Genie conversation
- Looks up by `thread_ts` and `slack_channel`

#### **Conversation Exists?**
- Decision point: Does this thread already have an active Genie conversation?
  - **YES** → Use existing conversation (go to "Create Message")
  - **NO** → Start new conversation (go to "Start Conversation")

---

### 3. **Genie Interaction** (Two Paths)

#### **Path A: New Conversation**

**Start Conversation**
- Creates a new Databricks Genie conversation
- Sends the user's question as the initial message
- Returns a new `conversation_id` and `message_id`

**Insert Conversation ID for thread_ts**
- Saves the new conversation mapping to PostgreSQL
- Links the Slack thread to the Genie conversation for future messages

**Set Fields for New Conversation**
- Prepares `message_id` and `conversation_id` for polling

#### **Path B: Existing Conversation**

**Create Message**
- Adds a new message to an existing Genie conversation
- Continues the conversation thread in Genie
- Returns a new `message_id` for this question

**Set Fields for Existing Conversation**
- Prepares `message_id` and `conversation_id` for polling

---

### 4. **Response Polling Loop** (Wait for Genie)

#### **Get Message**
- Polls Databricks Genie to check if the response is ready
- Retrieves the current status and content of the message

#### **Check response status**
- Checks if the message status is `COMPLETED`
  - **YES** → Process the response
  - **NO** → Wait and retry

#### **Code** (Backoff Logic)
- Calculates exponential backoff wait times:
  - First 24 attempts: 5 seconds each (2 minutes total)
  - After that: exponentially increases (5s, 10s, 20s, 40s, 60s max)
  - Tracks total elapsed time
  - Sets `status: "timeout"` if over 10 minutes

#### **Check Timeout**
- Checks if polling has exceeded the 10-minute limit
  - **NO timeout** → Continue waiting
  - **TIMEOUT** → Exit loop with error

#### **Wait Backoff**
- Pauses execution for the calculated backoff time
- Then loops back to "Get Message" to check again

---

### 5. **Response Processing** (Format & Send)

#### **Check response content**
- Determines what type of response Genie provided by checking if a query attachment exists
- **TRUE (Has query attachment)** → Path A: Process SQL query results
- **FALSE (Text only)** → Path B: Use text response as-is

---

#### **Path A: Query Results with Data** (TRUE branch)

This path processes SQL query results into both a CSV file and natural language summary:

**Get Query Results**
- Retrieves the full SQL query results from Genie
- Gets the data table, schema, and metadata

**Flatten Genie Query Results**
- JavaScript code node that transforms the raw query response into a structured format
- Extracts column names from the schema
- Maps data arrays to JSON objects with column names as keys
- Prepares data for CSV conversion

**Genie Query Results CSV Conversion**
- Converts the flattened JSON data to CSV format
- Creates a binary file named `GenieResults.csv`
- Includes header row with column names

**Summarize query attachment**
- Uses an AI language model (Llama 4 Maverick) to summarize the results
- Prompt includes:
  - The original question
  - Query description
  - Actual query results (JSON)
- AI generates a natural language summary instead of a raw table
- Has error handling (`onError: continueErrorOutput`)

**Databricks Chat Model**
- Powers the AI summarization
- Uses Databricks' hosted Llama model (`databricks-llama-4-maverick`)

**→ Leads to:** "Format LLM Message" then "Send Genie Response with File"

---

#### **Path B: Text-Only Response** (FALSE branch)

This path handles simple text responses from Genie that don't involve data queries:

**Format text output**
- Extracts the text content from Genie's response (`attachments[0].text.content`)
- Converts markdown formatting for Slack:
  - Changes bullet points (`-` to `*`)
  - Removes double asterisks (`**` to `*`)
- No AI summarization or CSV generation needed

**→ Leads to:** "Send Genie Text Response"

---

### 6. **Slack Response** (Send Back to User)

Both paths eventually send a response back to Slack, but with different formatting and content:

#### **Formatting Nodes:**

**Format LLM Message** (from Path A - Query Results)
- Formats the AI-generated summary for Slack:
  - Converts markdown list format (`-` to `*`)
  - Removes double asterisks (converts `**` to `*`)
  - Creates a link to the Genie conversation in Databricks

**Format Error Message** (from LLM errors)
- Handles error scenarios (like LLM summarization failures)
- Formats error messages similarly for Slack

**Format text output** (from Path B - Text Response)
- Already formatted in Path B section above
- Applies same markdown conversions for consistency

---

#### **Slack Send Nodes:**

**Send Genie Response with File** (Path A - Query Results)
- Sends query results with CSV attachment back to Slack
- Includes the AI-generated summary as the message
- Attaches the raw query results as a CSV file (`GenieResults.csv`) for reference
- Posts in the same thread as the original question
- Includes a link to the Genie Space conversation
- **Triggered by:** Query results that have been processed and summarized

**Send Genie Text Response** (Path B - Text Only)
- Sends plain text responses back to Slack
- Posts in the same thread as the original question
- Uses Slack markdown formatting for rich text
- No file attachment needed
- **Triggered by:** Simple text responses from Genie

---

## Error Handling Workflow

A separate **Slack Genie Error Workflow** is configured to handle any errors that occur during execution.

### Error Workflow Nodes:

#### **Error Trigger**
- Automatically triggered when the main workflow encounters an error
- Receives error details and execution context

#### **Get execution data**
- Retrieves the full execution data from the failed workflow run
- Uses n8n API to fetch:
  - Original Slack event data (channel, thread)
  - Error message and stack trace
  - Execution ID and workflow context

#### **Send a message**
- Sends the error message back to the user in Slack
- Posts in the same thread where the original question was asked
- Extracts channel and thread information from the failed execution
- Ensures users are notified of failures instead of being left waiting

This error workflow ensures **graceful degradation** - if something goes wrong, users get immediate feedback in Slack rather than the bot silently failing.

---

## Key Features

### ✅ **Thread Continuity**
The workflow maintains conversation context by linking Slack threads to Genie conversations in PostgreSQL, allowing multi-turn conversations. Each Slack thread maps to a single Genie conversation, so follow-up questions maintain context.

### ✅ **Smart Polling**
Uses exponential backoff to efficiently wait for responses without overwhelming the API:
- Quick checks initially (every 5 seconds for the first 24 attempts)
- Gradually slows down for longer queries (exponential backoff capped at 60 seconds)
- 10-minute maximum timeout to prevent infinite loops

### ✅ **Natural Language Responses**
SQL query results are automatically:
1. Converted to CSV format for download
2. Summarized into easy-to-read prose by an LLM
3. Posted to Slack with both the summary and CSV attachment

### ✅ **Error Handling**
Gracefully handles timeouts and errors, sending user-friendly messages back to Slack through a dedicated error workflow.

### ✅ **Robust Error Recovery**
A separate error workflow catches any failures and ensures users are always notified in Slack, preventing silent failures.

### ✅ **Link to Source**
Every response includes a direct link back to the Genie conversation in Databricks for users who want to explore further or verify results.

---

## Data Flow Summary

### Main Workflow
```
Slack Mention
    ↓
Check if thread has existing Genie conversation
    ↓
    ├─ New: Start Genie conversation → Save to DB
    └─ Existing: Add message to conversation
    ↓
Poll Genie every few seconds (exponential backoff)
    ↓
Response ready? (Check response status = COMPLETED)
    ↓
Check response content type
    ↓
    ├─ PATH A: Has query attachment (SQL results)
    │   ↓
    │   Get Query Results → Flatten Data → Convert to CSV
    │   ↓
    │   Summarize with AI (LLM)
    │   ↓
    │   Format LLM Message
    │   ↓
    │   Send to Slack with CSV attachment + summary + link
    │
    └─ PATH B: Text only response
        ↓
        Format text output
        ↓
        Send text message to Slack (no attachment)
```

### Error Workflow
```
Error occurs in main workflow
    ↓
Error Trigger activated
    ↓
Fetch execution data (channel, thread info)
    ↓
Send error message to user in Slack thread
```

---

## Technical Components

- **Slack Socket Mode**: Real-time event streaming from Slack (no webhook polling required)
- **Databricks Genie**: AI-powered data analytics assistant that converts natural language to SQL
- **PostgreSQL**: Stores conversation thread mappings between Slack threads and Genie conversations
- **n8n**: Workflow automation orchestration platform
- **Llama 4 Maverick**: Large language model for natural language summarization of query results
- **CSV Conversion**: Binary file generation for downloadable query results

---

## Workflow Files

- **Main Workflow**: `slack_genie_integration_workflow.json`
  - Handles all Slack-to-Genie interactions
  
- **Error Workflow**: `Slack Genie Error Workflow.json`
  - Catches and reports errors from the main workflow

