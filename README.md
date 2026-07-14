# FASTRAK Respond.io External AI Gateway

This project is ready to deploy. You do not edit the JavaScript code.

You only provide:

1. An OpenAI API key **or** a Groq API key for Llama.
2. A long random gateway secret.
3. The hosted gateway URL inside Respond.io.
4. Respond.io dynamic variables selected from its variable picker.

## Recommended provider

Use OpenAI first:

- `AI_PROVIDER=openai`
- `OPENAI_API_KEY=your key`
- `OPENAI_MODEL=gpt-5-mini`

Llama alternative through Groq:

- `AI_PROVIDER=groq`
- `GROQ_API_KEY=your key`
- `GROQ_MODEL=llama-3.3-70b-versatile`

## Fastest deployment: Render Blueprint

1. Create a private GitHub repository.
2. Upload every file from this folder to the repository root.
3. In Render, select **New > Blueprint**.
4. Connect the repository.
5. Render reads `render.yaml` and creates the web service.
6. Enter the API key requested by Render.
7. Render automatically generates `GATEWAY_SECRET`. Open the service's **Environment** page and copy that generated value for Respond.io.
8. Wait for deployment to complete.
9. Open `https://YOUR-SERVICE.onrender.com/health` and confirm that it says `healthy`.

For Llama, change only the Render environment values:

- Set `AI_PROVIDER` to `groq`.
- Enter `GROQ_API_KEY`.
- Leave `GROQ_MODEL` at its supplied default or replace it with a currently supported Groq Llama model ID.

## Local test

1. Copy `.env.example` to `.env`.
2. Fill in the key and `GATEWAY_SECRET`.
3. Run:

```bash
npm install
npm start
```

Test with Thunder Client:

- Method: `POST`
- URL: `http://localhost:3000/respondio/ai`
- Header: `Authorization: Bearer YOUR_GATEWAY_SECRET`
- Header: `Content-Type: application/json`
- Body:

```json
{
  "message": "Hi, do you have the FTS 32-inch TV and how many are left?",
  "first_name": "Thabo",
  "contact_id": "test-001",
  "channel": "WhatsApp",
  "language": "auto",
  "conversation_context": ""
}
```

## Respond.io Workflow configuration

The gateway is designed for Respond.io's **HTTP Request** Workflow step. Respond.io then sends the returned text through its native **Send a Message** step. No Respond.io Developer API endpoint needs to be coded.

### A. Create the Workflow

1. Open Respond.io.
2. Go to **Workflows**.
3. Click **Add Workflow**.
4. Name it `FASTRAK External AI - OpenAI or Llama`.
5. Choose the trigger that runs for a new incoming customer message in your workspace.
6. Add any channel, business-hours or human-assignment conditions you require before the AI step.
7. Add an **HTTP Request** step.

### B. Configure HTTP Request

- Method: `POST`
- URL: `https://YOUR-RENDER-SERVICE.onrender.com/respondio/ai`
- Headers:
  - `Authorization` = `Bearer YOUR_GATEWAY_SECRET`
  - `Content-Type` = `application/json`

Body type: JSON. Paste this structure, then replace only the placeholder strings by selecting the matching Respond.io variables from the variable picker:

```json
{
  "message": "REPLACE_THIS_WITH_RESPONDIO_LAST_INCOMING_MESSAGE_VARIABLE",
  "first_name": "REPLACE_THIS_WITH_RESPONDIO_FIRST_NAME_VARIABLE",
  "contact_id": "REPLACE_THIS_WITH_RESPONDIO_CONTACT_ID_VARIABLE",
  "channel": "REPLACE_THIS_WITH_RESPONDIO_CHANNEL_VARIABLE",
  "language": "auto",
  "conversation_context": ""
}
```

The exact variable labels can differ by Respond.io trigger and workspace version. Select them using Respond.io's picker rather than typing variable syntax manually.

### C. Save response fields as Workflow variables

Save these JSON response keys:

- `$.reply` as `external_ai_reply`
- `$.handoff_required` as `external_ai_handoff`
- `$.handoff_reason` as `external_ai_handoff_reason`
- `$.category` as `external_ai_category`
- Save HTTP response status as `external_ai_status`

### D. Add success/failure routing

Add a Branch step:

- Success condition: `external_ai_status` equals `200`.
- Failure branch: send a short fallback message and assign the conversation to a human team.

On the success branch, add another Branch:

- If `external_ai_handoff` equals `true`:
  1. Send `external_ai_reply`.
  2. Add a tag such as `AI - Human Handoff`.
  3. Assign to your Customer Support team.
  4. Add an internal comment containing `external_ai_handoff_reason`.
  5. End the AI workflow.

- Otherwise:
  1. Add **Send a Message**.
  2. Message content: the `external_ai_reply` variable.
  3. End the Workflow or return to your message-waiting structure.

## Important operating rule

Do not allow this Workflow to run while a conversation is actively assigned to a human agent. Add a Branch before the HTTP Request step that checks your handoff tag or assignment status and ends the Workflow when human handling is active.

## Why the gateway returns HTTP 200 on provider failures

Respond.io's HTTP Request step has a 10-second timeout. The gateway uses an 8.5-second maximum and returns a safe human-handoff message instead of allowing the Workflow to fail silently.

## Security

- Never commit `.env`.
- Never place the AI API key directly in Respond.io.
- Respond.io receives only the gateway secret.
- Rotate keys immediately if exposed.
- Use a paid Render service for production to avoid cold-start delays.
