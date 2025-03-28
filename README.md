# Conversational Blockchain Interaction Framework - Developer Documentation

**Version:** 1.0
**Last Updated:** March 27, 2025

## 1. Introduction / Overview

### 1.1. Purpose

This framework enables users to initiate potentially complex blockchain transactions through a natural language conversation with an AI Chatbot (like the Claude Desktop App). It decouples the user interaction and intent parsing from the secure transaction signing process. The goal is to provide a user-friendly conversational entry point for Web3 actions while ensuring transaction security remains firmly in the user's control via their existing blockchain wallet and a dedicated signing gateway DApp.

### 1.2. Key Features

* **Conversational Interface:** Users interact via natural language with an LLM-powered application (Application C).
* **Standardized LLM Integration:** Leverages the **Model Context Protocol (MCP)** for communication between the LLM application (acting as MCP Client/Host) and the backend transaction builder (Server A, acting as MCP Server).
* **Off-Chain Transaction Building:** Complex transaction logic (e.g., finding the best swap route) is handled by a dedicated backend (Server A).
* **Secure Signing:** The user signs the transaction using their own wallet (e.g., MetaMask) through a minimal, dedicated web application (DApp B - Signing Gateway), ensuring private keys never leave the user's control.
* **User Confirmation:** The user performs the final review and approval of transaction details directly within their trusted wallet interface.
* **Transaction Tracking & Notification:** The backend monitors the submitted transaction and notifies the conversational interface upon completion or failure.

### 1.3. Target Use Cases

* Performing DeFi actions (swaps, liquidity provision) via chatbot commands.
* Minting, listing, or managing NFTs conversationally.
* Interacting with DAO proposals (e.g., voting) through an LLM interface.
* Any blockchain interaction where a conversational starting point is desired, but secure, user-controlled signing is paramount.

### 1.4. High-Level Architecture

```mermaid
graph LR
    User -- Interacts --> AppC[Application C (Chatbot / MCP Client)];
    AppC -- 1. Call MCP Tool --> ServerA[Server A (Tx Builder / MCP Server / API)];
    ServerA -- 2. Return Signing URL --> AppC;
    AppC -- 3. Display URL --> User;
    User -- 4. Clicks URL --> DAppB[DApp B (Signing Gateway)];
    DAppB -- 5. Fetch Tx Data (API) --> ServerA;
    ServerA -- 6. Return Tx Data --> DAppB;
    DAppB -- 7. Request Signature --> Wallet[(User's Wallet)];
    Wallet -- 8. User Confirms --> DAppB;
    Wallet -- 9. Broadcast Tx --> Blockchain((Blockchain));
    DAppB -- 10. Submit Tx Hash (API) --> ServerA;
    ServerA -- 11. Monitor Tx --> Blockchain;
    ServerA -- 12. Send Notification (Webhook) --> AppCBackend(App C Backend);
    AppCBackend -- 13. Update UI --> AppC;
    AppC -- 14. Show Final Status --> User;

    style Wallet fill:#f9f,stroke:#333,stroke-width:2px;
    style Blockchain fill:#ccf,stroke:#333,stroke-width:2px;

## 2. Core Components

### 2.1. Application C (e.g., Claude Desktop App)

* **Role:** User Interface (Conversational), Intent Parser, MCP Client/Host.
* **Responsibilities:**
    * Engage the user in a natural language conversation.
    * Parse the user's intent to determine the desired blockchain action and parameters.
    * Act as an MCP Client to connect to Server A (MCP Server).
    * Call the appropriate MCP Tool on Server A (e.g., `build_blockchain_transaction`), passing required parameters.
    * Receive the signing URL from the MCP Tool result.
    * Display the signing URL clearly to the user, instructing them to proceed with signing in DApp B.
    * Receive asynchronous status updates (via its backend) about the transaction's final state (success/failure) and present them to the user.

### 2.2. Server A (Transaction Builder & Orchestrator)

* **Role:** Backend Logic, Transaction Construction, State Management, MCP Server, Standard API Provider.
* **Responsibilities:**
    * **MCP Server Interface:**
        * Implement the MCP Server protocol (likely via stdio if used with Claude Desktop).
        * Expose one or more MCP Tools (e.g., `build_blockchain_transaction`) with a defined input schema (JSON Schema) reflecting required parameters (action type, amounts, tokens, target contracts, user address, etc.).
        * Implement the logic for the exposed MCP Tool(s):
            * Fetch necessary on-chain or off-chain data (e.g., prices, allowances, nonces, gas estimates, contract addresses).
            * Construct the complete, unsigned blockchain transaction object (`txObject` - including `to`, `value`, `data`, `gasLimit`, `maxFeePerGas`, `maxPriorityFeePerGas`, `nonce`, `chainId`).
            * Persist the `txObject` temporarily (e.g., in a database or cache) associated with a unique, securely generated transaction identifier (`txid`). **Ensure `txid` is not easily guessable.**
            * Return a securely constructed URL pointing to DApp B, containing the `txid` (e.g., `https://<dapp-b-domain>/sign?txid=<txid>`) as the result of the MCP Tool call.
    * **Standard API Interface:**
        * Provide an authenticated/authorized API endpoint (e.g., `GET /api/transactions/{txid}`) for DApp B to securely retrieve the `txObject` associated with a given `txid`.
        * Provide an API endpoint (e.g., `POST /api/transactions/{txid}/submitted`) for DApp B (or potentially the wallet, though less common) to submit the transaction hash (`txHash`) once the transaction is broadcasted.
        * (Optional) Provide an API endpoint (e.g., `GET /api/transactions/{txid}/status`) for status polling.
    * **Transaction Tracking:**
        * Using the received `txHash`, monitor the corresponding blockchain for transaction confirmation, finality, and status (success or failure/revert).
    * **Notification:**
        * Upon detecting transaction finality, send an asynchronous notification (preferably via a secure **Webhook**) to Application C's backend infrastructure, including the `txid`, final status, `txHash`, and any relevant details (e.g., amount received in a swap).

### 2.3. DApp B (Signing Gateway)

* **Role:** Minimal, Secure Web Interface for Wallet Connection and Transaction Signing Facilitation. **Crucially, DApp B does NOT build or modify transactions.**
* **Responsibilities:**
    * Be served over **HTTPS**.
    * Extract the `txid` from the incoming URL query parameter (e.g., from `/sign?txid=...`).
    * Make a secure API call to Server A's `GET /api/transactions/{txid}` endpoint to fetch the pre-built `txObject`. Handle potential errors (invalid `txid`, expired request, server error).
    * Prompt the user to connect their Web3 wallet (e.g., using libraries like ethers.js, viem, web3-react, wagmi) via the standard browser provider (`window.ethereum`) or WalletConnect.
    * Once the wallet is connected and the `txObject` is fetched, use the wallet provider's API (e.g., `ethereum.request({ method: 'eth_sendTransaction', params: [txObject] })`) to pass the **exact, unaltered `txObject`** to the user's wallet.
    * Clearly indicate to the user that they must **verify the transaction details within their wallet extension/app**.
    * (Optional but Recommended) Listen for the `txHash` returned by the wallet provider after the user signs and the wallet broadcasts.
    * (Optional but Recommended) Send the received `txHash` back to Server A via the `POST /api/transactions/{txid}/submitted` API endpoint.
    * Display feedback to the user (e.g., "Transaction submitted", "Waiting for confirmation", link to block explorer using the `txHash`). Minimize complex UI elements; focus on facilitating the signing step.

### 2.4. User's Wallet (e.g., MetaMask, Trust Wallet)

* **Role:** Secure Private Key Storage, User Confirmation Interface, Transaction Signer & Broadcaster.
* **Responsibilities:**
    * Securely manage the user's private keys.
    * Intercept the `eth_sendTransaction` request initiated by DApp B.
    * **Critically:** Parse and display the details of the `txObject` (recipient address, value/amount, Gas fee estimate, data/function call decoded if possible) in its **own trusted UI**.
    * Await explicit user confirmation (approval/signing) or rejection.
    * If approved, sign the transaction using the user's private key.
    * Broadcast the signed transaction to the appropriate blockchain network.
    * Return the `txHash` to the requesting DApp (DApp B).

### 2.5. Blockchain

* **Role:** Decentralized Ledger, Smart Contract Execution Environment.
* **Responsibilities:**
    * Receive signed transactions broadcasted by wallets.
    * Validate transactions (signature, nonce, balance).
    * Include valid transactions in blocks according to network consensus rules.
    * Execute the state changes defined in the transaction (e.g., calling smart contract functions).
    * Provide a public, verifiable history of transactions and state.

## 3. User Experience (UX) Flow

This outlines the typical journey for a user interacting with the framework:

1.  **Initiate:** User expresses their intent in natural language within Application C (e.g., "Swap 1 ETH for DAI", "Mint the latest NFT drop").
2.  **Process:** Application C confirms understanding (if necessary) and communicates with Server A via MCP in the background.
3.  **Redirect to Sign:** Application C presents a message like: *"Alright, I've prepared the transaction for you. Please click this secure link to review and approve it in our Signing Gateway (DApp B) using your wallet: [link to DApp B with txid]"*.
4.  **Open Signing Gateway:** User clicks the link, which opens DApp B in their default web browser.
5.  **Connect Wallet:** DApp B prompts the user to connect their blockchain wallet (if not already connected from a previous session on that DApp). The user approves the connection in their wallet.
6.  **Wallet Prompt:** Immediately after connection (or after DApp B fetches the data), the user's **wallet extension/application** automatically pops up, displaying the full details of the transaction prepared by Server A.
7.  **USER VERIFICATION (Critical Step):** The user **must carefully review** the transaction details presented *inside their wallet's trusted interface*. This includes:
    * The destination address (is it the expected contract?).
    * The amount of native currency being sent (ETH, MATIC, etc.).
    * The estimated Gas fee.
    * The function being called and its parameters (if the wallet can decode them).
8.  **Approve/Reject:**
    * If the details look correct, the user clicks "Confirm" / "Sign" in their wallet.
    * If anything looks suspicious or incorrect, the user clicks "Reject" in their wallet.
9.  **Feedback in DApp B:** After the user confirms in the wallet, DApp B's interface updates to show "Transaction submitted! Waiting for confirmation..." and may display the `txHash` with a link to a block explorer. The user can close DApp B at this point or wait.
10. **Switch Back:** The user navigates back to Application C.
11. **Final Notification:** Once Server A detects the transaction is finalized on the blockchain, Application C receives an update (via its backend) and displays the final status message in the conversation: *"Success! Your transaction (Hash: [txHash]) is confirmed."* or *"Unfortunately, your transaction failed. Reason: [error message from blockchain if available]"*.

## 4. Technical Workflow (Sequence Diagram)

```mermaid
sequenceDiagram
    participant User
    participant AppC_UI [App C UI (e.g., Claude Desktop)]
    participant AppC_Backend [App C Backend]
    participant ServerA [Server A (MCP Server / API)]
    participant DAppB [DApp B (Signing Gateway)]
    participant Wallet [(User's Wallet)]
    participant Blockchain ((Blockchain))

    User->>AppC_UI: 1. Input Intent (e.g., "Swap 1 ETH")
    AppC_UI->>AppC_Backend: 2. Send Parsed Intent
    AppC_Backend->>ServerA: 3. Call MCP Tool (`build_tx`, params)
    ServerA->>ServerA: 4. Build txObject, Store, Gen txid
    ServerA-->>AppC_Backend: 5. Return Result (URL: `.../sign?txid=...`)
    AppC_Backend-->>AppC_UI: 6. Send URL to UI
    AppC_UI->>User: 7. Display URL/Link
    User->>DAppB: 8. Click Link (Opens DApp B)
    DAppB->>ServerA: 9. GET /api/transactions/{txid}
    ServerA-->>DAppB: 10. Return txObject
    DAppB->>Wallet: 11. Request Wallet Connection (if needed)
    Wallet-->>DAppB: 12. Wallet Connected
    DAppB->>Wallet: 13. ethereum.request('eth_sendTransaction', [txObject])
    Wallet->>User: 14. Show Tx Details in Wallet UI
    User->>Wallet: 15. Review & Confirm/Sign
    Wallet->>Blockchain: 16. Broadcast Signed Transaction
    Wallet-->>DAppB: 17. Return txHash
    DAppB->>ServerA: 18. POST /api/transactions/{txid}/submitted (body: {txHash})
    ServerA-->>DAppB: 19. Ack Submission (Optional)
    DAppB->>User: 20. Show "Submitted..." (Optional Explorer Link)

    ServerA->>Blockchain: 21. Monitor Transaction Status (using txHash)
    Blockchain-->>ServerA: 22. Transaction Confirmed / Failed
    ServerA->>AppC_Backend: 23. POST /webhook/c-callback (body: {txid, status, txHash, ...})
    AppC_Backend->>AppC_Backend: 24. Process Notification, Update Session State
    AppC_Backend-->>AppC_UI: 25. Push Update to UI
    AppC_UI->>User: 26. Display Final Status (Success/Failure)

## 5. Implementation Details & APIs

### 5.1. Server A - MCP Interface

* **Protocol:** Model Context Protocol (MCP)
* **Transport:** Typically `stdio` when launched by local clients like Claude Desktop. Can also support HTTP/SSE for remote clients.
* **Required Tool(s):** Define at least one tool, e.g., `build_blockchain_transaction`.
* **Example `inputSchema` (JSON Schema) for `build_blockchain_transaction` Tool:**
    ```json
    {
      "type": "object",
      "properties": {
        "action_type": {
          "type": "string",
          "description": "The type of blockchain action requested.",
          "enum": ["swap", "mint_nft", "dao_vote", "transfer"]
        },
        "parameters": {
          "type": "object",
          "description": "Parameters specific to the action_type.",
          "properties": {
            "from_token": { "type": "string", "description": "Symbol or address of token to send (for swap/transfer)" },
            "to_token": { "type": "string", "description": "Symbol or address of token to receive (for swap)" },
            "amount": { "type": "string", "description": "Amount to send/swap (in smallest unit or human-readable)" },
            "recipient": { "type": "string", "description": "Recipient address (for transfer)" },
            "contract_address": { "type": "string", "description": "Target contract address (for mint/vote)" },
            "token_id": { "type": "string", "description": "NFT Token ID (for mint)" },
            "proposal_id": { "type": "string", "description": "DAO Proposal ID (for vote)" },
            "vote_option": { "type": "boolean", "description": "Vote choice (true/false, yes/no)" },
            "slippage_tolerance": { "type": "number", "description": "Optional: Slippage tolerance in basis points (e.g., 50 for 0.5%)" }
          },
          "required": [
             // Define required params based on action_type logic within the tool
          ]
        },
        "user_address": {
          "type": "string",
          "description": "The blockchain address of the user initiating the action."
        },
         "chain_id": {
            "type": "integer",
            "description": "The target blockchain network ID (e.g., 1 for Ethereum Mainnet)."
         }
      },
      "required": ["action_type", "parameters", "user_address", "chain_id"]
    }
    ```
* **Tool Result:** A single string containing the fully formed URL for DApp B, e.g., `"https://sign.example.com/sign?txid=abc123xyz789"`.
### 5.2. Server A - Standard API

* **Authentication/Authorization:** All endpoints should be protected. Consider API keys, JWT tokens, or session-based auth depending on the context, especially for the `GET /api/transactions/{txid}` endpoint. Rate limiting is also advised.
* **`GET /api/transactions/{txid}`**
    * **Path Parameter:** `{txid}` - The unique transaction identifier.
    * **Success Response (200 OK):**
        ```json
        {
          "to": "0x...",       // Target address (contract or EOA)
          "value": "0x...",    // Amount of native currency (in Wei, hex encoded)
          "data": "0x...",    // Encoded function call and parameters
          "gasLimit": "0x...", // Estimated gas limit (hex encoded)
          "maxFeePerGas": "0x...", // EIP-1559 max fee (hex encoded)
          "maxPriorityFeePerGas": "0x...", // EIP-1559 priority fee (hex encoded)
          "nonce": "0x...",    // User's nonce for this transaction (hex encoded)
          "chainId": "0x..."   // Chain ID (hex encoded)
          // DO NOT include sensitive data or the private key here!
        }
        ```
    * **Error Responses:** `404 Not Found` (invalid/expired `txid`), `401 Unauthorized`, `403 Forbidden`, `500 Internal Server Error`.
* **`POST /api/transactions/{txid}/submitted`**
    * **Path Parameter:** `{txid}` - The unique transaction identifier.
    * **Request Body:**
        ```json
        {
          "txHash": "0x[64 hex characters]"
        }
        ```
    * **Success Response (200 OK or 204 No Content):** Confirmation that the hash was received.
    * **Error Responses:** `400 Bad Request` (invalid `txHash` format), `404 Not Found`, `409 Conflict` (hash already submitted?), `500 Internal Server Error`.
* **Webhook to App C Backend (Example: `POST /app-c-webhook/transaction-update`)**
    * **Security:** Must be secured using a mechanism like HMAC signature verification based on a shared secret, or IP whitelisting.
    * **Request Body:**
        ```json
        {
          "txid": "abc123xyz789",
          "status": "success", // or "failed"
          "txHash": "0x...",
          "blockNumber": 1234567, // Optional
          "gasUsed": "0x...", // Optional (hex encoded)
          "details": { // Optional: Application-specific details
            "action_type": "swap",
            "amount_out": "123456789000000000000" // Example: Amount received
          },
          "error_message": null // or "Transaction reverted: Reason..." if failed
        }
        ```

### 5.3. DApp B (Signing Gateway)

* **Technology Stack:** Standard web technologies (HTML, CSS, JavaScript). Frontend framework (React, Vue, Svelte, etc.) recommended. Web3 library (ethers.js, viem, web3.js) essential.
* **Core Logic:**
    1.  On page load, parse `txid` from URL.
    2.  Display "Connecting to wallet..." and initiate wallet connection request (e.g., `ethereum.request({ method: 'eth_requestAccounts' })`).
    3.  Once connected, display "Fetching transaction details...". Call Server A's `GET /api/transactions/{txid}`.
    4.  On receiving the `txObject`, immediately call `ethereum.request({ method: 'eth_sendTransaction', params: [txObject] })`.
    5.  Handle the promise returned by `eth_sendTransaction`:
        * On success (user signed, wallet broadcasted), receive `txHash`. Display "Transaction submitted..." and optionally call Server A's `POST /api/transactions/{txid}/submitted` with the `txHash`. Provide link to block explorer.
        * On error (user rejected, RPC error, etc.), display an appropriate error message.
* **UI:** Keep it minimal and focused. Clearly state its purpose (signing gateway for transactions prepared by [Your Service Name]). Emphasize that final verification happens *in the wallet*.

### 5.4. Application C

* **LLM:** Capable of parsing natural language and extracting structured parameters.
* **MCP Client:** Use official MCP SDKs (Python, TypeScript, Java, Kotlin, C# as available) to implement the client logic for connecting to Server A and calling its Tool(s). Refer to MCP documentation for SDK usage.
* **UI:** Present the signing URL clearly. Handle the asynchronous nature of the final result (update the conversation when the webhook is received by the backend).
* **Backend:** Needs an endpoint to receive webhook notifications from Server A. Logic to map the notification back to the correct user/conversation and trigger a UI update.

## 6. Security Considerations

* **Trust Chain:** The user must trust Application C (and its backend), Server A, DApp B, and their Wallet software. Each component introduces a potential point of failure or attack.
* **Server A:**
    * **API Security:** Securely authenticate and authorize all API endpoints (`GET /transactions/{txid}`, `POST /transactions/{txid}/submitted`). Use HTTPS. Implement rate limiting.
    * **TxID Security:** Generate unpredictable `txid`s. Expire `txid`s after a reasonable time or after first use to prevent replay.
    * **Input Validation:** Rigorously validate all parameters received via MCP Tool calls and API requests.
    * **Webhook Security:** Secure the webhook endpoint on App C's backend and verify incoming requests from Server A (e.g., using HMAC signatures with a shared secret).
    * **Internal Logic:** Ensure the transaction building logic is correct and doesn't introduce vulnerabilities (e.g., reentrancy if interacting with own contracts, incorrect slippage calculations).
* **DApp B:**
    * **Serve over HTTPS.**
    * **Minimal Logic:** Its *only* job related to the transaction is fetching the pre-built `txObject` and passing it *unaltered* to the wallet. **It MUST NOT construct, modify, or enrich the transaction data.**
    * **Prevent XSS:** Ensure user inputs (if any, ideally none) are properly sanitized to prevent cross-site scripting.
    * **Source Clarity:** Clearly state that the transaction was prepared elsewhere (by the service running Server A).
* **Wallet:**
    * The wallet's UI is the **single source of truth** for the user before signing. Wallets must accurately parse and display transaction details.
* **User Responsibility:** **This is paramount.** Users MUST be educated to:
    * **ALWAYS carefully verify transaction details (recipient, amount, gas, data) within their wallet's pop-up before clicking "Confirm".** Do not blindly trust what DApp B or Application C displays about the transaction details before the wallet prompt.
    * Be wary of phishing links pretending to be DApp B or messages from Application C. Verify URLs.
    * Understand the permissions they grant when connecting their wallet.

## 7. Getting Started / Developer Setup

1.  **Server A:**
    * Choose a language/framework (Node.js, Python, Go, Java, etc.).
    * Implement the MCP Server logic using an official MCP SDK, defining the necessary Tool(s).
    * Implement the standard web API (e.g., using Express, Flask, Gin, Spring Boot).
    * Implement blockchain interaction logic (using libraries like ethers.js, web3.py, etc., to fetch nonce, estimate gas, monitor transactions).
    * Set up necessary infrastructure (database/cache for `txObject` storage, blockchain node access).
    * Configure environment variables (blockchain node URL, API keys, webhook secrets, etc.).
2.  **DApp B:**
    * Set up a standard web frontend project.
    * Integrate a Web3 library (ethers.js, viem).
    * Implement logic to parse URL, call Server A's API, connect wallet, and request signing.
    * Deploy to a static hosting provider with HTTPS.
3.  **Application C (e.g., Claude Desktop):**
    * If using Claude Desktop, configure `claude_desktop_config.json` to launch Server A (likely via `stdio`). Specify the command and arguments to run Server A. Ensure the `command` uses an absolute path or is globally available in the PATH. Refer to MCP documentation for Claude Desktop configuration.
    * If building a custom App C, implement the MCP Client logic using an SDK. Implement the backend webhook receiver.

## 8. Troubleshooting

* **MCP Connection Issues (C <-> A):** Check Claude Desktop logs (`~/Library/Logs/Claude/mcp*.log` or `%APPDATA%\Claude\logs\mcp*.log`). Use the [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) tool to test Server A independently. Verify `claude_desktop_config.json` syntax and paths. Ensure Server A starts correctly and listens on stdio if configured for it.
* **API Errors (B <-> A):** Check network requests in DApp B's browser developer console. Check Server A's API logs for errors (4xx, 5xx). Verify API authentication/authorization.
* **Wallet Not Prompting:** Ensure DApp B is correctly calling `ethereum.request({ method: 'eth_sendTransaction', params: [txObject] })`. Check browser console for errors. Ensure the `txObject` fetched from Server A is valid.
* **Transaction Fails/Reverts:** Check the transaction on a block explorer using the `txHash`. Common reasons: insufficient gas, insufficient balance, smart contract logic revert (e.g., slippage too high, allowance not set), incorrect nonce. Server A should ideally simulate the transaction before storing the `txObject` to catch some errors early.
* **Webhook Not Received (A -> C Backend):** Check Server A's logs for webhook sending errors. Check App C's backend logs for incoming webhook errors (e.g., signature validation failure). Verify network connectivity and firewall rules.

---



