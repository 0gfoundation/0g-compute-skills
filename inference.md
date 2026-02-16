# Inference Reference

Complete guide for using 0G Compute Network inference services.

## SDK Integration Patterns

### Initialize Broker

#### Node.js Environment

```typescript
import { ethers } from "ethers";
import { createZGComputeNetworkBroker } from "@0glabs/0g-serving-broker";

const RPC_URL = process.env.NODE_ENV === 'production'
  ? "https://evmrpc.0g.ai"  // Mainnet
  : "https://evmrpc-testnet.0g.ai";  // Testnet

const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);
const broker = await createZGComputeNetworkBroker(wallet);
```

#### Browser Environment

```typescript
import { BrowserProvider } from "ethers";
import { createZGComputeNetworkBroker } from "@0glabs/0g-serving-broker";

// Check if MetaMask is installed
if (typeof window.ethereum === "undefined") {
  throw new Error("Please install MetaMask");
}

const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const broker = await createZGComputeNetworkBroker(signer);
```

### Browser Polyfills

When using the SDK in browser environments, you need polyfills:

```bash
pnpm add -D vite-plugin-node-polyfills
```

```javascript
// vite.config.js
import { nodePolyfills } from 'vite-plugin-node-polyfills';

export default {
  plugins: [
    nodePolyfills({
      include: ['crypto', 'stream', 'util', 'buffer', 'process'],
      globals: { Buffer: true, global: true, process: true }
    })
  ]
};
```

## Service Discovery

```typescript
// List all available services
const services = await broker.inference.listService();

// List services with detailed health metrics
// Returns ServiceWithDetail[] with real-time uptime and latency information
const servicesWithDetail = await broker.inference.listServiceWithDetail();
servicesWithDetail.forEach(service => {
  console.log(`Provider: ${service.provider}`);
  console.log(`Service Type: ${service.serviceType}`);
  console.log(`Model: ${service.model}`);

  // Health metrics from monitoring API (optional field)
  if (service.healthMetrics) {
    console.log(`  Status: ${service.healthMetrics.status}`);           // 'healthy' | 'warning' | 'critical' | 'unknown'
    console.log(`  Uptime: ${service.healthMetrics.uptime}%`);          // Success rate percentage (0-100)
    console.log(`  Avg Response Time: ${service.healthMetrics.avgResponseTime}ms`); // Average latency in milliseconds
    console.log(`  Last Check: ${service.healthMetrics.lastCheck}`);    // ISO timestamp of last health check
  }
});

// Filter by service type
const chatbotServices = services.filter(s => s.serviceType === 'chatbot');
const imageServices = services.filter(s => s.serviceType === 'text-to-image');
const speechServices = services.filter(s => s.serviceType === 'speech-to-text');

// Pagination and filtering options
const servicesPage2 = await broker.inference.listService(50, 50); // offset=50, limit=50
const withUnacknowledged = await broker.inference.listService(0, 50, true); // include unacknowledged providers
```

### Provider Selection Based on Health Metrics

Use `listServiceWithDetail()` to make informed provider selection decisions:

```typescript
// Get services with health metrics
const servicesWithDetail = await broker.inference.listServiceWithDetail();

// Filter for healthy providers with good performance
const healthyProviders = servicesWithDetail.filter(service => {
  return service.healthMetrics &&
         service.healthMetrics.status === 'healthy' &&
         service.healthMetrics.uptime >= 95 &&  // At least 95% uptime
         service.healthMetrics.avgResponseTime < 1000;  // Less than 1 second latency
});

// Sort by uptime (descending)
const sortedByUptime = [...servicesWithDetail]
  .filter(s => s.healthMetrics)
  .sort((a, b) => b.healthMetrics!.uptime - a.healthMetrics!.uptime);

// Sort by response time (ascending - fastest first)
const sortedByLatency = [...servicesWithDetail]
  .filter(s => s.healthMetrics)
  .sort((a, b) => a.healthMetrics!.avgResponseTime - b.healthMetrics!.avgResponseTime);

// Get the most reliable provider for a specific service type
const bestChatbotProvider = servicesWithDetail
  .filter(s => s.serviceType === 'chatbot' && s.healthMetrics)
  .sort((a, b) => {
    // Prioritize uptime, then latency
    if (a.healthMetrics!.uptime !== b.healthMetrics!.uptime) {
      return b.healthMetrics!.uptime - a.healthMetrics!.uptime;
    }
    return a.healthMetrics!.avgResponseTime - b.healthMetrics!.avgResponseTime;
  })[0];
```

## Account Management

```typescript
// Get account info
const account = await broker.ledger.getLedger();

// Deposit funds to main account
await broker.ledger.depositFund(10);

// Acknowledge provider (required before first use)
await broker.inference.acknowledgeProviderSigner(providerAddress);
```

## Chatbot Inference

### Standard Chat Completion

```typescript
const messages = [{ role: "user", content: "Hello!" }];

// Get service metadata
const { endpoint, model } = await broker.inference.getServiceMetadata(providerAddress);

// Generate auth headers
const headers = await broker.inference.getRequestHeaders(providerAddress);

// Make request
const response = await fetch(`${endpoint}/chat/completions`, {
  method: "POST",
  headers: { "Content-Type": "application/json", ...headers },
  body: JSON.stringify({ messages, model })
});

const data = await response.json();
const answer = data.choices[0].message.content;

// Process response for fee management and verification
let chatID = response.headers.get("ZG-Res-Key") || response.headers.get("zg-res-key");
if (!chatID) {
  chatID = data.id;  // Use completion.id from response body
}

if (chatID && data.usage) {
  await broker.inference.processResponse(
    providerAddress,
    chatID,
    JSON.stringify(data.usage)
  );
} else if (data.usage) {
  await broker.inference.processResponse(
    providerAddress,
    undefined,
    JSON.stringify(data.usage)
  );
}
```

### Streaming Chat Completion

```typescript
const response = await fetch(`${endpoint}/chat/completions`, {
  method: "POST",
  headers: { "Content-Type": "application/json", ...headers },
  body: JSON.stringify({
    messages,
    model,
    stream: true
  })
});

// For chatbot streaming: check headers first, then try stream data
let chatID = response.headers.get("ZG-Res-Key") || response.headers.get("zg-res-key");
let usage = null;
let streamChatID = null;

const decoder = new TextDecoder();
const reader = response.body.getReader();
let rawBody = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  rawBody += decoder.decode(value, { stream: true });

  // Process chunks in real-time
  console.log(decoder.decode(value, { stream: true }));
}

// Parse usage and chatID from stream
for (const line of rawBody.split('\n')) {
  const trimmed = line.trim();
  if (!trimmed || trimmed === 'data: [DONE]') continue;

  try {
    const jsonStr = trimmed.startsWith('data:') ? trimmed.slice(5).trim() : trimmed;
    const message = JSON.parse(jsonStr);

    // For chatbot, try to get ID from stream data (use completion.id)
    if (!streamChatID && message.id) {
      streamChatID = message.id;
    }
    if (message.usage) {
      usage = message.usage;
    }
  } catch {}
}

// Use chatID from header if available, otherwise use chatID from stream data
const finalChatID = chatID || streamChatID;

if (finalChatID) {
  await broker.inference.processResponse(
    providerAddress,
    finalChatID,
    JSON.stringify(usage || {})
  );
} else if (usage) {
  await broker.inference.processResponse(
    providerAddress,
    undefined,
    JSON.stringify(usage)
  );
}
```

## Text-to-Image Generation

```typescript
const prompt = "A cute baby sea otter playing in the water";

// Get service metadata
const { endpoint, model } = await broker.inference.getServiceMetadata(providerAddress);

const requestBody = {
  model,
  prompt,
  n: 1,
  size: "1024x1024"
};

// Generate auth headers
const headers = await broker.inference.getRequestHeaders(
  providerAddress,
  JSON.stringify(requestBody)
);

// Make request
const response = await fetch(`${endpoint}/images/generations`, {
  method: "POST",
  headers: { "Content-Type": "application/json", ...headers },
  body: JSON.stringify(requestBody)
});

const data = await response.json();
const imageUrl = data.data[0].url;

// Get chatID from response headers only
const chatID = response.headers.get("ZG-Res-Key") || response.headers.get("zg-res-key");

if (chatID) {
  const isValid = await broker.inference.processResponse(
    providerAddress,
    chatID
  );
  console.log("Response valid:", isValid);
} else {
  // Fallback: process without verification if no chatID
  await broker.inference.processResponse(
    providerAddress,
    undefined
  );
}
```

## Speech-to-Text Transcription

### Standard Transcription

```typescript
const formData = new FormData();
formData.append('file', audioFile); // audioFile is a File or Blob
formData.append('model', model);
formData.append('response_format', 'json');

// Get service metadata
const { endpoint, model } = await broker.inference.getServiceMetadata(providerAddress);

// Generate auth headers
const headers = await broker.inference.getRequestHeaders(providerAddress);

// Make request
const response = await fetch(`${endpoint}/audio/transcriptions`, {
  method: "POST",
  headers: { ...headers },
  body: formData
});

const data = await response.json();
const transcription = data.text;

// Get chatID from response headers
const chatID = response.headers.get("ZG-Res-Key") || response.headers.get("zg-res-key");

if (chatID && data.usage) {
  const isValid = await broker.inference.processResponse(
    providerAddress,
    chatID,
    JSON.stringify(data.usage)
  );
  console.log("Response valid:", isValid);
} else if (data.usage) {
  await broker.inference.processResponse(
    providerAddress,
    undefined,
    JSON.stringify(data.usage)
  );
}
```

### Streaming Transcription

```typescript
const response = await fetch(`${endpoint}/audio/transcriptions`, {
  method: "POST",
  headers: { ...headers },
  body: formData
});

// For speech-to-text streaming, get chatID from headers
const chatID = response.headers.get("ZG-Res-Key") || response.headers.get("zg-res-key");

let usage = null;
const decoder = new TextDecoder();
const reader = response.body.getReader();
let rawBody = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  rawBody += decoder.decode(value, { stream: true });
}

// Parse usage from stream data
for (const line of rawBody.split('\n')) {
  const trimmed = line.trim();
  if (!trimmed || trimmed === 'data: [DONE]') continue;

  try {
    const jsonStr = trimmed.startsWith('data:') ? trimmed.slice(5).trim() : trimmed;
    const message = JSON.parse(jsonStr);
    if (message.usage) {
      usage = message.usage;
    }
  } catch {}
}

// Process with chatID for verification if available
if (chatID) {
  const isValid = await broker.inference.processResponse(
    providerAddress,
    chatID,
    JSON.stringify(usage || {})
  );
  console.log("Audio streaming response valid:", isValid);
} else if (usage) {
  await broker.inference.processResponse(
    providerAddress,
    undefined,
    JSON.stringify(usage)
  );
}
```

## Direct API Usage (cURL)

### Get API Secret

```bash
0g-compute-cli inference get-secret --provider <PROVIDER_ADDRESS>
```

This generates a Bearer token: `app-sk-<SECRET>`

### Chatbot API

```bash
curl <service_url>/v1/proxy/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer app-sk-<YOUR_SECRET>" \
  -d '{
    "model": "<MODEL_NAME>",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "Hello!"
      }
    ]
  }'
```

### Text-to-Image API

```bash
curl <service_url>/v1/proxy/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer app-sk-<YOUR_SECRET>" \
  -d '{
    "model": "<MODEL_NAME>",
    "prompt": "A cute baby sea otter playing in the water",
    "n": 1,
    "size": "1024x1024"
  }'
```

### Speech-to-Text API

```bash
curl <service_url>/v1/proxy/audio/transcriptions \
  -H "Authorization: Bearer app-sk-<YOUR_SECRET>" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@audio.ogg" \
  -F "model=whisper-large-v3" \
  -F "response_format=json"
```

## Local Proxy Server

Run a local OpenAI-compatible server:

```bash
# Start server on port 3000 (default)
0g-compute-cli inference serve --provider <PROVIDER_ADDRESS>

# Custom port
0g-compute-cli inference serve --provider <PROVIDER_ADDRESS> --port 8080
```

Then use any OpenAI-compatible client to connect to `http://localhost:3000`.

## Python Examples

### Chatbot

```python
from openai import OpenAI

client = OpenAI(
    base_url=f"{service_url}/v1/proxy",
    api_key='app-sk-<YOUR_SECRET>'
)

completion = client.chat.completions.create(
    model=model,
    messages=[
        {
            'role': 'system',
            'content': 'You are a helpful assistant.'
        },
        {
            'role': 'user',
            'content': 'Hello!'
        }
    ]
)

print(completion.choices[0].message)
```

### Text-to-Image

```python
from openai import OpenAI

client = OpenAI(
    base_url=f"{service_url}/v1/proxy",
    api_key='app-sk-<YOUR_SECRET>'
)

response = client.images.generate(
    model=model,
    prompt='A cute baby sea otter playing in the water',
    n=1,
    size='1024x1024'
)

print(response.data)
```

### Speech-to-Text

```python
from openai import OpenAI

client = OpenAI(
    base_url=f"{service_url}/v1/proxy",
    api_key='app-sk-<YOUR_SECRET>'
)

with open('audio.ogg', 'rb') as audio_file:
    transcription = client.audio.transcriptions.create(
        file=audio_file,
        model='whisper-large-v3',
        response_format='json'
    )

print(transcription.text)
```

## Key Points

### Response Verification

- Always call `processResponse()` after receiving responses
- The SDK automatically handles fund transfers to prevent service interruptions
- For verifiable TEE services, the method also validates response integrity

### chatID Retrieval

**Principle**: Always prioritize `ZG-Res-Key` from response headers. Only use fallback methods when header is not present.

- **Chatbot**: First try `ZG-Res-Key` header, then check `data.id` (completion ID from response body) as fallback
- **Text-to-Image & Speech-to-Text**: Always get chatID from `ZG-Res-Key` response header
- **Streaming responses**:
  - **Chatbot streaming**: Check headers first, then try to get `id` from stream data as fallback
  - **Speech-to-text streaming**: Get chatID from `ZG-Res-Key` header immediately

### Usage Data

Usage data format varies by service type but typically includes token counts or request metrics.
