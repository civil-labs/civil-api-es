# civil-api-js
This package contains the generated Universal JavaScript (ESM) handlers and TypeScript definitions for the Civil API. It is built to work seamlessly with Node.js, modern web browsers, and bundlers like Vite, Webpack, or Next.js.

Installation
Install this package along with the required ConnectRPC peer dependencies for web clients:

```bash
npm install @civil-labs/civil-api-js @connectrpc/connect @connectrpc/connect-web
```
(Note: If you are building a Node.js backend instead of a web frontend, swap @connectrpc/connect-web for @connectrpc/connect-node).

## Quick Start
With Connect v2, both your data structures (Messages) and your service definitions are exported from the same file. You can import them directly from the package root.

Here is a basic example of how to set up the transport and make an API call from a frontend application:

```typescript
import { createClient } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web";

// 1. Import the service and request messages from the SDK
import { ParcelService, GetParcelRequest } from "@civil-labs/civil-api-js";

// 2. Configure the network transport 
const transport = createConnectTransport({
  baseUrl: "<target jurisdiction's endpoint>",
});

// 3. Create a type-safe client for the specific service
const client = createClient(ParcelService, transport);

// 4. Make an API call
async function fetchParcelData(parcelId: string) {
  try {
    const response = await client.getParcel(
      new GetParcelRequest({ id: parcelId })
    );
    console.log("Parcel Data:", response.parcel);
  } catch (error) {
    console.error("Failed to fetch parcel:", error);
  }
}
```

## Authentication (OIDC Bearer Tokens)
External requests to the Civil platform are secured using OIDC Bearer Tokens issued by the platform instance's identity provider service.

Because access tokens expire and refresh, the safest and most reliable way to handle this in ConnectRPC is by using an Interceptor. This ensures that every API call automatically fetches the most up-to-date token before leaving the browser.

```typescript
import { createClient } from "@connectrpc/connect";
import type { Interceptor } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web";
import { ParcelService, GetParcelRequest } from "@civil-labs/civil-api-js";

// 1. Define your token retrieval logic (depends on your OIDC client)
async function getValidAccessToken(): Promise<string | null> {
  // Example: return await authClient.getTokenSilently();
  return "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."; 
}

// 2. Create the auth interceptor
const authInterceptor: Interceptor = (next) => async (req) => {
  const token = await getValidAccessToken();
  
  if (token) {
    // Attach the Bearer token to the outgoing request headers
    req.header.set("Authorization", `Bearer ${token}`);
  }
  
  return await next(req); // Continue the request
};

// 3. Apply the interceptor to the transport
const transport = createConnectTransport({
  baseUrl: "<target jurisdiction's endpoint>",
  interceptors: [authInterceptor], // <-- Add it here
});

// 4. Create your client as usual
const client = createClient(ParcelService, transport);
```

### Handling 401 Unauthorized Errors
If the OIDC token expires and the backend rejects the request, the server will return a specific Connect error code: Code.Unauthenticated. This can be caughts globally to trigger a redirect to the identity provider's login page or kick off a silent token refresh.

```typescript
import { createClient, ConnectError, Code } from "@connectrpc/connect";
import type { Interceptor } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web";
import { ParcelService } from "@civil-labs/civil-api-js";

// 1. Create the Response Interceptor
const errorHandlingInterceptor: Interceptor = (next) => async (req) => {
  try {
    // Await the response from the server
    return await next(req);
  } catch (err) {
    // Check if the error is a recognized ConnectRPC error
    if (err instanceof ConnectError) {
      if (err.code === Code.Unauthenticated) {
        console.warn("Session expired or invalid token. Triggering logout...");
        
        // TODO: Add your global logout or refresh logic here
        // Example: window.location.href = '/login';
        // Example: await authProvider.refreshToken();
      }
      
      if (err.code === Code.PermissionDenied) {
        console.warn("User does not have access to this resource.");
      }
    }
    
    // Always re-throw the error so the specific component that made 
    // the API call can still catch it and show a local error message
    throw err;
  }
};

// 2. Add both interceptors to the transport array
const transport = createConnectTransport({
  baseUrl: "<target jurisdiction's endpoint>",
  interceptors: [
    authInterceptor,          // Attaches the token
    errorHandlingInterceptor  // Catches the errors
  ],
});

const client = createClient(ParcelService, transport);
```