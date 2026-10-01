# API Schemas: Sessions & Conversations

### RunSessionRequest
- **config** (-> SessionConfig): [required]
- **inputs** (array[-> SessionInput]): [required]

### RunSessionResponse
- **outputs** (array[-> SessionOutput])

### SessionConfig
- **inputAudioConfig** (-> InputAudioConfig)
- **outputAudioConfig** (-> OutputAudioConfig)
- **historicalContexts** (array[-> Message]): Override session history.
- **entryAgent** (string): Entry agent (default: root). Format: `projects/.../agents/{agent}`
- **timeZone** (string): User's IANA time zone.
- **useToolFakes** (boolean): Use fake tools.
- **deployment** (string): Deployment to use for the session.
- **remoteDialogflowQueryParameters** (object): `webhookHeaders`, `payload`, `endUserMetadata` sent to a remote Dialogflow agent when control is transferred to it.
- **enableTextStreaming** (boolean): Stream text outputs (text sessions via StreamRunSession or BidiRunSession only).
- **excludeDiagnosticInfo** (boolean): Exclude diagnostic info from session output. Missing diagnostics do not mean nothing ran server-side.

### SessionInput
- **text** (string): User text.
- **dtmf** (string): DTMF digits.
- **audio** (string): Audio data.
- **toolResponses** (-> ToolResponses): Client tool results.
- **image** (-> Image)
- **blob** (-> Blob)
- **variables** (object): Session variables, keyed by name. Only declared variables used by CES; unrecognized variables are still sent to a remote Dialogflow agent as session parameters.
- **event** (-> Event)
- **willContinue** (boolean): Bidi only. Marks a non-audio input as a fragment; processing waits for `willContinue=false`. Does not apply to audio or DTMF.

### SessionOutput
- **text** (string): Agent text.
- **audio** (string): Agent audio.
- **toolCalls** (-> ToolCalls): Tool requests for client.
- **endSession** (-> EndSession): Session ended.
- **payload** (object): Custom structured payload.
- **turnIndex** (integer): Turn number (from 1).
- **turnCompleted** (boolean): Agent finished this turn.

### EndSession
- **metadata** (object): Reason for ending.

### Message
- **role** (string): `user` or `agent`.
- **chunks** (array[-> Chunk])

### Chunk
- **text** (string)
- **transcript** (string): Audio transcript.
- **toolCall** (-> ToolCall)
- **toolResponse** (-> ToolResponse)
- **agentTransfer** (-> AgentTransfer)
- **updatedVariables** (object)

### Event
- **event** (string): [required] Event name.
- **variables** (object): Additional variables associated with the event.

### InputAudioConfig
- **audioEncoding** (enum: `LINEAR16` | `MULAW` | `ALAW`): [required]
- **sampleRateHertz** (integer): [required]

### OutputAudioConfig
- **audioEncoding** (enum: `LINEAR16` | `MULAW` | `ALAW`): [required]
- **sampleRateHertz** (integer): [required]

### Conversation
- **name** (string): Format: `projects/.../conversations/{conversation}`
- **turns** (array[-> ConversationTurn])
- **turnCount** (integer): Output only.
- **source** (enum: `LIVE` | `SIMULATOR` | `EVAL` | `AGENT_TOOL`): Output only. `AGENT_TOOL` is the separate session an agent-as-a-tool runs in.
- **inputTypes** (array[enum]): Output only. `INPUT_TYPE_TEXT`, `INPUT_TYPE_EVENT`, `INPUT_TYPE_AUDIO`, `INPUT_TYPE_IMAGE`, `INPUT_TYPE_BLOB`, `INPUT_TYPE_TOOL_RESPONSE`, ... Replaces deprecated `channelType`.
- **appVersion** (string): Output only. App version that processed the conversation.
- **deployment** (string): Output only.
- **entryAgent** (string): Output only.

### ConversationTurn
- **messages** (array[-> Message])
