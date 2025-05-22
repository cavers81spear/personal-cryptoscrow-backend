# CryptoEscrow Backend

[Download here](https://github.com/cavers81spear/personal-cryptoscrow-backend/releases)

## Overview

CryptoEscrow is a comprehensive backend system designed to power a secure, trustless escrow platform for cryptocurrency-based property or high-value asset transactions. Leveraging **Node.js**, **Express**, and **Firebase** (Firestore, Authentication, Storage), the backend is responsible for:

*   User identity management (authentication and basic profile data).
*   Secure API endpoints for all client-side operations.
*   Detailed deal (escrow transaction) creation, tracking, and lifecycle management.
*   Off-chain storage and management of deal conditions and related documents.
*   Interaction with the `PropertyEscrow.sol` Solidity smart contract for on-chain escrow logic.
*   Automated monitoring and execution of time-sensitive on-chain actions based on deal deadlines.

The `PropertyEscrow.sol` smart contract, intended for deployment on Ethereum-compatible networks (e.g., Sepolia testnet, Ethereum Mainnet), is the ultimate arbiter of funds, managing deposits, condition-based state transitions, dispute periods, and final fund disbursal or cancellation.

This document serves as a primary guide for frontend developers, outlining how to effectively integrate with the CryptoEscrow backend API and understand its core functionalities.

## Key Features & Frontend Interaction Points

-   **Secure User Authentication**:
    -   **Backend**: Integrates seamlessly with Firebase Authentication, supporting Email/Password and Google Sign-In methods. Manages user sessions and secures API endpoints using Firebase ID Tokens.
    -   **Frontend Interaction**:
        -   Use the Firebase Client SDK to implement user sign-up and sign-in flows.
        -   Upon successful authentication, the Firebase SDK provides an ID Token.
        -   This ID Token **MUST** be included in the `Authorization` header as a Bearer token for all requests to protected backend API routes (e.g., `Authorization: Bearer <YOUR_FIREBASE_ID_TOKEN>`).
        -   The frontend should also handle token refresh scenarios as managed by the Firebase SDK.

-   **Contact Management**:
    -   **Backend**: Provides functionality for users to build and manage a list of contacts, facilitating easier selection of parties when initiating new escrow deals. Manages invitations between users.
    -   **Frontend Interaction**:
        -   Develop UI components for sending contact invitations (e.g., by email).
        -   Display lists of pending invitations (both sent and received) with options to accept, decline, or cancel.
        -   Show an authenticated user's established contact list.
        -   Allow users to remove contacts.
        -   Integrate contact selection into the "new deal" creation form.

-   **Deal Lifecycle Management**:
    -   **Backend**: Orchestrates the entire lifecycle of an escrow deal.
        -   **Deal Creation**: Allows authenticated users to initiate new escrow deals, defining key parameters such as the parties involved, property/asset details, escrow amount, currency, and initial off-chain conditions. Can be configured to deploy a unique `PropertyEscrow` smart contract instance per deal via `contractDeployer.js`.
        -   **Off-Chain Condition Tracking**: Manages a list of user-defined conditions (e.g., title clearance, property inspection, document notarization) within Firestore. Tracks the fulfillment status of these conditions.
        -   **On-Chain State Synchronization**: Reflects the smart contract's state (e.g., `AWAITING_DEPOSIT`, `IN_FINAL_APPROVAL`, `IN_DISPUTE`) in Firestore for easy frontend access. Provides an endpoint for manual state synchronization if needed.
        -   **Automated Deadline Management**: A critical backend process (`scheduledJobs.js`) monitors smart contract deadlines (e.g., final approval window, dispute resolution period). It uses `blockchainService.js` to trigger necessary on-chain transactions (e.g., `releaseFundsAfterApprovalPeriod`, `cancelEscrowAndRefundBuyer`) and `databaseService.js` to update Firestore records accordingly.
    -   **Frontend Interaction**:
        -   Implement comprehensive forms to capture all required details for deal creation. Upon successful creation, clearly display the unique deal ID and its initial status.
        -   Provide UI for the buyer to review and mark their off-chain conditions as fulfilled. Display the current status of all conditions for both parties.
        -   Clearly display the current on-chain synchronized status of the deal. The UI must be reactive to status changes, whether driven by direct user actions, actions of the other party, or automated backend processes.
        -   **Crucially, use Firestore real-time listeners** on deal documents to reflect changes to status, conditions, timeline events, and deadlines promptly in the UI without requiring manual polling.
        -   Inform users about upcoming deadlines and the implications of automated processes.

-   **File Management**:
    -   **Backend**: Facilitates secure upload, storage, and download of documents related to escrow deals using Firebase Storage. Associates files with specific deals and users.
    -   **Frontend Interaction**:
        -   Implement UI for file uploads (e.g., using `FormData` and `multipart/form-data` requests), allowing users to attach relevant documents to a deal.
        -   Display a list of files associated with a deal, showing metadata (name, type, upload date) and providing secure download links.

-   **Real-Time Updates**:
    -   **Backend**: Leverages Firebase Firestore's real-time capabilities extensively.
    -   **Frontend Interaction**: This is a cornerstone of a smooth user experience. The frontend **MUST** subscribe to real-time updates for relevant Firestore documents and collections (especially individual deal documents and lists of deals/contacts/invitations). This ensures the UI always reflects the current state of data without manual polling.

## Tech Stack

-   **Backend**: Node.js, Express.js
-   **Database & Authentication**: Firebase Admin SDK (Firestore, Firebase Authentication, Firebase Storage)
-   **Smart Contract Environment**: Solidity, Hardhat (for development, testing, deployment of `PropertyEscrow.sol` in `src/contract`)
-   **Blockchain Interaction**: Ethers.js (v6)
-   **Scheduled Jobs**: `node-cron`
-   **Email Notifications**: Nodemailer (or similar, for contact invitations, deal notifications - if implemented)
-   **Testing**:
    -   Backend: Jest, Supertest
    -   Smart Contract: Hardhat, Chai, Ethers.js

## Getting Started (Frontend Perspective)

1.  **Backend Base URL**: All API endpoints are relative to a base URL, which will be provided (e.g., `https://your-cryptoescrow-backend.com/api` or `http://localhost:3000/api` for local development).
2.  **Authentication Setup**:
    -   Integrate the Firebase Client SDK into your frontend application. Configure it with the Firebase project credentials matching the backend.
    -   Implement user sign-up and sign-in UI flows (e.g., forms for email/password, buttons for Google Sign-In).
    -   After a user successfully signs in, the Firebase Client SDK will provide an ID Token.
    -   **For every subsequent request to protected backend API endpoints, you must include this ID Token in the `Authorization` HTTP header, prefixed with "Bearer "**:
        `Authorization: Bearer <YOUR_FIREBASE_ID_TOKEN>`
    -   The Firebase Client SDK typically handles automatic token refresh. Ensure your HTTP client interceptors correctly use the latest token.
3.  **CORS (Cross-Origin Resource Sharing)**: The backend needs to be configured with your frontend application's URL in its `FRONTEND_URL` environment variable. This allows your frontend to make requests to the backend API from a different origin. Without this, browsers will block such requests.

## API Endpoints for Frontend Integration

The backend exposes a RESTful API. All routes requiring authentication expect a Firebase ID Token. Standard HTTP status codes are utilized (e.g., `200 OK`, `201 Created`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `500 Internal Server Error`). Error responses generally follow the format: `{ "error": "A descriptive error message for the user or for debugging." }`.

API routes are modularized within `src/api/routes/`. It's recommended to also review the specific README files within those subdirectories for more granular details.

### Authentication (`/auth`)
*(Managed in `src/api/routes/auth/loginSignUp.js`)*

These endpoints are used to register and sign in users. Note that while the backend handles these, the primary session management and ID token retrieval should be done via the Firebase Client SDK on the frontend.

#### 1. Sign Up with Email & Password
-   **Endpoint**: `POST /auth/signUpEmailPass`
-   **Description**: Registers a new user with the system using their email and password. The backend creates the user in Firebase Authentication.
-   **Request Body**:
    ```json
    {
      "email": "newuser@example.com",
      "password": "SecurePassword123!",
      "displayName": "John Doe" // Optional: Firebase uses this for user's profile display name
    }
    ```
-   **Success Response (201 Created)**:
    ```json
    {
      "message": "User created successfully. Please verify your email if applicable.", // Message may vary
      "uid": "firebaseUserIdGeneratedByFirebase"
    }
    ```
-   **Error Responses**:
    -   `400 Bad Request`: Missing `email` or `password`.
    -   `401 Unauthorized` (or similar like `409 Conflict`): Email already in use (`auth/email-already-in-use`).
    -   Other Firebase-specific auth errors.
-   **Frontend Actions**:
    -   Provide a sign-up form to collect email, password, and (optionally) display name.
    -   Consider client-side validation for password strength and email format.
    -   Upon form submission, call this backend endpoint.
    -   If successful, the Firebase Client SDK on the frontend (if user also signed in via client SDK) should reflect the new user state. The frontend might then redirect to a "please verify email" page or directly to the app's main interface.

#### 2. Sign In with Email & Password
-   **Endpoint**: `POST /auth/signInEmailPass`
-   **Description**: Signs in an existing user with their email and password.
-   **Request Body**:
    ```json
    {
      "email": "existinguser@example.com",
      "password": "TheirPassword123"
    }
    ```
-   **Success Response (200 OK)**:
    ```json
    {
      "message": "User signed in successfully.",
      "uid": "firebaseUserIdOfExistingUser" // Or email, needs confirmation from backend implementation
      // Backend might optionally return the ID token here, but best practice is for
      // frontend to get it directly from Firebase Client SDK after a client-side signIn.
    }
    ```
-   **Error Responses**:
    -   `400 Bad Request`: Missing `email` or `password`.
    -   `401 Unauthorized`: Invalid credentials (`auth/wrong-password`, `auth/user-not-found`).
-   **Frontend Actions**:
    -   Provide a sign-in form.
    -   Call this endpoint. More commonly, the frontend uses the Firebase Client SDK's `signInWithEmailAndPassword` method directly. If using this backend endpoint, ensure the Firebase Client SDK is also made aware of the session.
    -   On success, store the ID token (if returned, or get from SDK) and navigate to the authenticated part of the application.
    -   Handle sign-in errors by displaying appropriate messages.

#### 3. Sign In/Up with Google
-   **Endpoint**: `POST /auth/signInGoogle`
-   **Description**: Authenticates a user using a Google ID token obtained from a Google Sign-In flow on the frontend. If the user is new, an account is created.
-   **Request Body**:
    ```json
    {
      "idToken": "googleIdTokenStringObtainedFromFrontendGoogleSignIn"
    }
    ```
-   **Success Response (200 OK)**:
    ```json
    {
      "message": "User authenticated successfully with Google.",
      "uid": "firebaseUserId",
      "email": "user@example.com",
      "displayName": "John Doe",
      "isNewUser": false, // Boolean indicating if this was a first-time sign-in for this Google account
      "isAdmin": true // or false, specific to this application's logic
    }
    ```
-   **Error Responses**:
    -   `400 Bad Request`: Missing `idToken`.
    -   `401 Unauthorized`: Invalid or expired Google ID token.
    -   `403 Forbidden`: If backend has specific whitelisting and this Google account is not authorized.
-   **Frontend Actions**:
    -   Implement Google Sign-In using the Firebase Client SDK (recommended) or Google Identity Services.
    -   After the user signs in with Google on the frontend, retrieve the Google ID token.
    -   Send this token to the `/auth/signInGoogle` backend endpoint.
    -   The Firebase Client SDK on the frontend should already reflect the authenticated state. This backend call is often for the backend to perform its own registration/verification steps.

### Contacts (`/contact`)
*(Managed in `src/api/routes/contact/contactRoutes.js`)*

_All endpoints require `Authorization: Bearer <ID_TOKEN>` header._

#### 1. Send Contact Invitation
-   **Endpoint**: `POST /contact/invite`
-   **Description**: Sends a contact invitation from the authenticated user to another person via their email.
-   **Request Body**:
    ```json
    {
      "recipientEmail": "friend@example.com"
    }
    ```
-   **Success Response (200 OK)**: `{ "message": "Invitation sent successfully." }` (May include invitation ID).
-   **Frontend Actions**: Allow user to input an email address. Handle success/error messages. Update UI if displaying pending sent invitations.

#### 2. List Pending Invitations
-   **Endpoint**: `GET /contact/pending`
-   **Description**: Retrieves a list of contact invitations that are pending for the authenticated user (i.e., invitations they have received).
-   **Success Response (200 OK)**:
    ```json
    [
      {
        "invitationId": "invitationDocId1",
        "senderEmail": "sender@example.com",
        "senderId": "senderFirebaseUid",
        "sentAt": "2023-10-26T10:00:00.000Z" // ISO 8601 Timestamp
      }
      // ... more invitations
    ]
    ```
-   **Frontend Actions**: Display these invitations, providing options to accept or decline each one.

#### 3. Respond to Invitation
-   **Endpoint**: `POST /contact/response`
-   **Description**: Allows the authenticated user to accept or decline a received contact invitation.
-   **Request Body**:
    ```json
    {
      "invitationId": "invitationDocId1",
      "action": "accept" // or "decline"
    }
    ```
-   **Success Response (200 OK)**: `{ "message": "Invitation <accepted/declined> successfully." }`.
-   **Frontend Actions**: Remove the invitation from the pending list. If accepted, add the sender to the user's contact list (or refresh the contact list).

#### 4. Get User's Contact List
-   **Endpoint**: `GET /contact/contacts`
-   **Description**: Retrieves the list of established contacts for the authenticated user.
-   **Success Response (200 OK)**:
    ```json
    [
      {
        "contactId": "contactRelationshipDocId1", // ID of the contact relationship document
        "userId": "friendFirebaseUid1",           // Firebase UID of the contact
        "email": "friend1@example.com",
        "displayName": "Friend One",
        "addedAt": "2023-10-25T10:00:00.000Z"  // ISO 8601 Timestamp
      }
      // ... more contacts
    ]
    ```
-   **Frontend Actions**: Display the contacts. This list can be used to populate "other party" fields when creating a new deal.

#### 5. Remove a Contact
-   **Endpoint**: `DELETE /contact/contacts/:contactId`
-   **Description**: Removes a contact from the authenticated user's contact list.
-   **URL Parameter**: `:contactId` is the ID of the contact relationship (e.g., `contactRelationshipDocId1` from the GET /contacts response).
-   **Success Response (200 OK)**: `{ "message": "Contact removed successfully." }`.
-   **Frontend Actions**: Update the displayed contact list.

### Deals (`/deals`) - Managing Escrow Transactions
*(Managed in `src/api/routes/transaction/transactionRoutes.js`)*

_All endpoints require `Authorization: Bearer <ID_TOKEN>` header._

#### 1. Create a New Escrow Deal
-   **Endpoint**: `POST /deals/create`
-   **Description**: Initiates a new escrow deal, creating a record in Firestore and potentially deploying a dedicated smart contract.
-   **Request Body Example**:
    ```json
    {
      "initiatedBy": "BUYER", // "BUYER" or "SELLER" - who is filling out this form
      "propertyAddress": "123 Main St, Anytown, USA", // Or other asset identifier
      "amount": "1.5", // Amount in ETH (backend converts to Wei for contract)
      "currency": "ETH", // Could be extended for other ERC20s if contract supports
      "otherPartyEmail": "seller@example.com", // Email of the other party
      "buyerWalletAddress": "0xBuyerWalletAddress...", // Buyer's Ethereum wallet address
      "sellerWalletAddress": "0xSellerWalletAddress...", // Seller's Ethereum wallet address
      "initialConditions": [ // Off-chain conditions defined by the initiator
        { "id": "cond-title", "type": "TITLE_DEED", "description": "Clear title deed verified by buyer", "fulfilled": false },
        { "id": "cond-inspection", "type": "INSPECTION", "description": "Property inspection passed and approved by buyer", "fulfilled": false }
      ],
      "deployToNetwork": "sepolia" // Optional: specify network (e.g., "sepolia", "mainnet")
                                 // Backend defaults if not provided and supports multiple.
    }
    ```
-   **Success Response (201 Created)**:
    ```json
    {
      "message": "Deal created successfully and contract deployment initiated (if applicable).",
      "transactionId": "firestoreDealIdGeneratedByBackend", // Crucial ID for Firestore listeners
      "smartContractAddress": "0xDeployedContractAddressOrNull", // Address if deployed, null otherwise or if deployment is async
      "status": "AWAITING_CONDITION_SETUP", // Or a relevant initial status from the contract/backend logic
      // ...other initial deal details like parties' UIDs, full condition objects with IDs, etc.
    }
    ```
-   **Frontend Actions**:
    -   Provide a comprehensive form to capture all deal parameters. Use contact list for easy "other party" selection.
    -   Validate inputs client-side (e.g., wallet address format, amount).
    -   On success, store the `transactionId`. **Immediately set up a Firestore real-time listener** for this `transactionId` to receive live updates.
    -   Display the initial deal status and details. Inform the user about next steps (e.g., waiting for other party, depositing funds).

#### 2. Retrieve Deal Details
-   **Endpoint**: `GET /deals/:transactionId`
-   **Description**: Fetches the complete current details of a specific escrow deal from Firestore.
-   **URL Parameter**: `:transactionId` is the Firestore document ID of the deal.
-   **Success Response (200 OK)**: A full deal object.
    ```json
    {
      "id": "firestoreDealId",
      "initiatedBy": "BUYER",
      "propertyAddress": "123 Main St...",
      "amount": "1.5", // As string
      "currency": "ETH",
      "buyerId": "buyerFirebaseUid",
      "sellerId": "sellerFirebaseUid",
      "buyerWalletAddress": "0x...",
      "sellerWalletAddress": "0x...",
      "status": "IN_FINAL_APPROVAL", // Current on-chain synchronized status
      "smartContractAddress": "0xDeployedContractAddress",
      "conditions": [
        { "id": "cond-title", "type": "TITLE_DEED", "description": "Clear title deed verified by buyer", "fulfilled": true, "fulfilledBy": "buyerId", "fulfilledAt": "timestamp" },
        { "id": "cond-inspection", "type": "INSPECTION", "description": "Property inspection passed and approved by buyer", "fulfilled": false }
      ],
      "timeline": [
        { "event": "Deal created", "timestamp": "...", "actorId": "buyerFirebaseUid", "actorRole": "BUYER" },
        { "event": "Funds deposited by buyer.", "timestamp": "...", "actorId": "buyerFirebaseUid", "actorRole": "BUYER", "transactionHash": "0xOnChainTxHash"}
        // ... more events: condition fulfillment, dispute raised, funds released, etc.
      ],
      "finalApprovalDeadlineBackend": "timestamp", // Backend calculated, reflects on-chain if applicable
      "disputeResolutionDeadlineBackend": "timestamp", // Backend calculated
      "createdAt": "timestamp",
      "updatedAt": "timestamp"
      // ... any other relevant fields
    }
    ```
-   **Frontend Actions**: Primarily, the frontend should rely on its Firestore real-time listener for the deal document rather than calling this endpoint repeatedly. This endpoint can be useful for an initial load if a listener isn't immediately available or as a fallback.

#### 3. List User's Deals
-   **Endpoint**: `GET /deals`
-   **Description**: Retrieves a paginated list of deals where the authenticated user is either the buyer or the seller.
-   **Query Parameters (Optional for Pagination)**:
    -   `?lastVisibleId=<firestore_document_id_of_last_deal_in_previous_page>`
    -   `?limit=<number_of_deals_per_page>` (e.g., 10)
-   **Success Response (200 OK)**:
    ```json
    {
      "deals": [
        // ...array of deal summary objects (subset of full deal details)
        {
          "id": "firestoreDealId1",
          "propertyAddress": "123 Main St...",
          "amount": "1.5",
          "currency": "ETH",
          "status": "IN_FINAL_APPROVAL",
          "otherPartyEmail": "seller@example.com", // Or buyer, depending on who the current user is
          "role": "BUYER" // User's role in this specific deal
        }
      ],
      "nextPageToken": "firestore_document_id_of_last_deal_in_current_page_or_null" // Use as lastVisibleId for next query
    }
    ```
-   **Frontend Actions**: Display deals in a list or dashboard. Implement "load more" or pagination controls using `nextPageToken`. Firestore real-time listeners can also be used on queries for live updates to the list.

#### 4. Buyer Updates Off-Chain Condition Status
-   **Endpoint**: `PUT /deals/:transactionId/conditions/:conditionId/buyer-review`
-   **Description**: Allows the buyer in a deal to mark one of their off-chain conditions as fulfilled or unfulfilled.
-   **URL Parameters**: `:transactionId`, `:conditionId` (ID of the condition within the deal's `conditions` array).
-   **Request Body**:
    ```json
    {
      "fulfilled": true // or false
    }
    ```
-   **Success Response (200 OK)**: `{ "message": "Condition status updated successfully." }` (The deal document in Firestore will be updated, triggering real-time listeners).
-   **Frontend Actions**: Provide checkboxes or buttons for the buyer to update condition statuses. The UI should react to the Firestore update. If all buyer conditions are met and funds are deposited, the backend/contract logic might automatically transition the deal state (e.g., to `READY_FOR_FINAL_APPROVAL`), which the frontend will see via its listener.

#### 5. Sync Backend with Smart Contract State (Optional User Action)
-   **Endpoint**: `PUT /deals/:transactionId/sync-status`
-   **Description**: Manually triggers the backend to query the smart contract for its current state and update the deal's status in Firestore if there's a discrepancy. Useful if the frontend suspects a temporary disconnect between off-chain and on-chain views.
-   **URL Parameter**: `:transactionId`.
-   **Success Response (200 OK)**: `{ "message": "Deal status synchronization initiated/completed.", "updatedStatus": "ACTUAL_STATUS_FROM_CHAIN", "previousStatus": "STATUS_IN_FIRESTORE_BEFORE_SYNC" }`.
-   **Frontend Actions**: Could be a "Refresh Deal Status" button. The UI will update based on Firestore listener upon successful sync.

### Files (`/files`)
*(Managed in `src/api/routes/database/fileUploadDownload.js` - Note: this route path might be `/api/database/...` or `/api/files/...` depending on main router setup)*

_All endpoints require `Authorization: Bearer <ID_TOKEN>` header._

#### 1. Upload Deal-Related File
-   **Endpoint**: `POST /files/upload`
-   **Request Type**: `multipart/form-data`
-   **Description**: Uploads a file associated with a specific deal.
-   **Form Data Fields**:
    -   `dealId` (string): The Firestore ID of the deal.
    -   `file` (file object): The actual file data.
    -   `fileType` (string, optional): User-defined type or category, e.g., "INSPECTION_REPORT", "TITLE_DEED", "AGREEMENT".
-   **Success Response (201 Created)**:
    ```json
    {
      "message": "File uploaded successfully.",
      "fileId": "generatedFileIdInFirestore", // ID of the file metadata document
      "fileName": "originalUploadedFileName.pdf",
      "fileURL": "https://firebasestorage.googleapis.com/...", // Signed URL or public URL for download
      "filePath": "deals/dealId/originalUploadedFileName.pdf", // Path in Firebase Storage
      "uploadedAt": "timestamp"
    }
    ```
-   **Frontend Actions**: Use a file input element. Construct `FormData` object. Handle upload progress if possible. On success, update the deal's file list in the UI.

#### 2. List File Metadata for User's Deals (or a specific deal)
-   **Endpoint**: `GET /files/my-deals` (or potentially `GET /deals/:transactionId/files`)
-   **Description**: Retrieves metadata for files associated with deals the current user is part of, or for a specific deal.
-   **Success Response (200 OK)**:
    ```json
    [
      {
        "fileId": "fileId1",
        "dealId": "dealIdAssociatedWithFile",
        "fileName": "document1.pdf",
        "fileType": "AGREEMENT",
        "uploadedBy": "uploaderFirebaseUid",
        "uploadedAt": "timestamp",
        "fileURL": "https://firebasestorage.googleapis.com/..."
      }
      // ... more file metadata objects
    ]
    ```
-   **Frontend Actions**: Display a list of files, often grouped by deal, with links to download.

#### 3. Download a Specific File
-   **Endpoint**: `GET /files/download/:dealId/:fileId`
-   **Description**: Provides a way to download a specific file. The backend handles permissions.
-   **URL Parameters**: `:dealId`, `:fileId`.
-   **Success Response**: The file itself (e.g., `Content-Type: application/pdf`). The browser will typically trigger a download.
-   **Frontend Actions**: Provide a download link/button for each file. This can be a direct `<a>` tag pointing to this endpoint or a programmatic fetch that handles the file stream for download.

### Health Check (`/health`)
*(Managed in `src/api/routes/health/health.js`)*

#### 1. Verify Backend Connectivity
-   **Endpoint**: `GET /health`
-   **Description**: A public endpoint to check the operational status of the backend.
-   **Success Response (200 OK)**:
    ```json
    {
      "status": "healthy",
      "message": "Backend services are operational.",
      "timestamp": "ISO_DATE_STRING",
      "firestoreConnected": true // Example check
    }
    ```
-   **Frontend Actions**: Generally not called by the main user-facing UI. Useful for DevOps, monitoring, or a status page.

## Smart Contract States & Frontend Implications

The `PropertyEscrow.sol` contract dictates the on-chain lifecycle. The backend synchronizes this state to Firestore (in the `deal.status` field). The frontend **MUST** reactively display user-friendly interpretations of these states and guide users on appropriate actions.

Key States (example names, actual may vary):

-   `AWAITING_CONDITION_SETUP` / `PENDING_BUYER_SETUP`: Deal created, buyer (or initiator) might need to finalize off-chain conditions or review terms.
-   `AWAITING_DEPOSIT`: All initial setup complete. Waiting for the buyer to deposit the escrow amount into the smart contract.
    -   **Frontend**: Guide buyer to make the on-chain deposit. Display clear instructions and potentially link to a wallet.
-   `AWAITING_FULFILLMENT` / `DEPOSIT_RECEIVED`: Funds are in escrow. Buyer now needs to mark their off-chain conditions as fulfilled via the backend API.
    -   **Frontend**: Enable UI for buyer to mark conditions. Show progress to both parties.
-   `READY_FOR_FINAL_APPROVAL` / `CONDITIONS_MET`: Buyer has marked all conditions as fulfilled. The deal is ready for the final approval period to begin. Either party might need to trigger this on-chain or via an API call.
    -   **Frontend**: Indicate this state. Explain the upcoming approval period.
-   `IN_FINAL_APPROVAL`: A fixed-duration countdown (e.g., 48 hours) has started. During this time, the buyer can raise a dispute if they find issues. If no dispute is raised and the timer expires, funds can be released to the seller.
    -   **Frontend**: Clearly display the countdown. Provide a prominent option for the buyer to raise a dispute (which would involve an on-chain transaction).
-   `IN_DISPUTE`: The buyer has formally raised a dispute on-chain. A longer fixed-duration countdown (e.g., 7 days) begins for resolution (e.g., buyer re-fulfilling conditions, mutual agreement).
    -   **Frontend**: Clearly indicate "Dispute Active." Display the dispute resolution deadline. Guide users on potential actions.
-   `COMPLETED` / `FUNDS_RELEASED`: The deal has concluded successfully; funds released to the seller (either automatically after `IN_FINAL_APPROVAL` or after dispute resolution).
    -   **Frontend**: Congratulatory message. Show transaction hash for fund release. Deal is closed.
-   `CANCELLED` / `REFUNDED`: Escrow cancelled. Funds (if deposited) returned to the buyer. This can happen due to mutual agreement, expired dispute deadline without resolution, or other cancellation conditions in the contract.
    -   **Frontend**: Inform users of cancellation. Show transaction hash for refund if applicable. Deal is closed.

**It is paramount that the frontend uses Firestore real-time listeners to observe `deal.status` and relevant deadline fields (`finalApprovalDeadlineBackend`, `disputeResolutionDeadlineBackend`) to provide a dynamic, accurate, and timely UI reflecting the true state of the escrow.**

## Automated Backend Processes (`scheduledJobs.js`)

The frontend should be aware that certain state transitions can happen **automatically** due to backend cron jobs:

1.  **Post-Final Approval Release**: If a deal is in `IN_FINAL_APPROVAL` and its `finalApprovalDeadlineBackend` passes without a dispute, the backend job will call `blockchainService.triggerReleaseAfterApproval`.
2.  **Post-Dispute Cancellation/Refund**: If a deal is in `IN_DISPUTE` and its `disputeResolutionDeadlineBackend` passes without resolution, the backend job will call `blockchainService.triggerCancelAfterDisputeDeadline` (which typically refunds the buyer).

These automated actions result in on-chain transactions, and the backend updates the deal's status and timeline in Firestore. The frontend, through its real-time listeners, will see these changes and should update the UI accordingly, informing the user that an automated process has occurred.

## Environment Variables (Recap for Frontend Context)

-   `FRONTEND_URL`: Crucial for CORS. The backend uses this to allow requests from your frontend's domain.
-   Firebase Client SDK configuration keys (API Key, Auth Domain, etc.): The frontend will need these to initialize the Firebase SDK. They must match the Firebase project used by the backend.

## Further Development & Considerations

-   **Detailed API Specification**: For very complex interactions, an OpenAPI (Swagger) specification could be generated by or requested from the backend team for precise request/response schemas and model definitions.
-   **Error Handling Granularity**: Frontend should gracefully handle various HTTP error codes and display user-friendly messages. Discuss specific error codes or error message patterns with the backend team for robust error handling.
-   **WebSockets (Alternative Real-time)**: While Firestore provides excellent real-time data synchronization, if there are needs for push notifications for events not directly tied to Firestore document changes (e.g., a generic "you have a new message" notification), WebSockets could be a future addition. For deal data, Firestore is generally sufficient.
-   **Internationalization (i18n)**: If the application supports multiple languages, API error messages and any descriptive text from the backend might need to be code-based or localized.
-   **API Versioning**: As the API evolves, versioning (e.g., `/api/v1/...`) might be introduced.

