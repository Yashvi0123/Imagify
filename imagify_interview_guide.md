# Imagify — Complete Interview Guide (Final Year Project)

## 1. Project Overview (Elevator Pitch)
**Imagify** is a full-stack, software-as-a-service (SaaS) web application designed to allow users to generate high-quality AI images dynamically from textual prompts. Using a secure credit-based monetization model, users receive a complimentary credit allotment upon signup and can purchase credit packages through multiple payment gateways once their balance is depleted.

* **Problem Statement:** Running cutting-edge AI image generation models locally is computationally expensive, requires high-end GPUs, and involves complex terminal configurations. Moreover, developers often struggle to build secure SaaS boilerplates that combine API security (hiding keys), state management, and double-payment gateway logic.
* **Solution:** Imagify addresses this by hosting a secure Node.js/Express API that proxies client requests to AI text-to-image engines (Clipdrop API) while masking API keys. It handles user authentication, tracks credit balances dynamically, handles real-time payment transactions via Stripe and Razorpay, and renders a highly responsive, animated frontend in React.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (React SPA - Vite)              │
│  Home Page │ Image Generator Workspace │ Buy Credits Page   │
│  React (Context API) + Tailwind CSS + Framer Motion         │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP REST Requests (Axios) + JWT
┌────────────────────────▼────────────────────────────────────┐
│                 BACKEND (Express.js on Node.js)             │
│  Auth Middleware │ User Controller │ Image Controller      │
│  Security: JWT Verification │ Bcrypt Password Hashing       │
└──────┬─────────────────────┬──────────────────────────┬─────┘
       │ Mongoose ODM        │ HTTP API Calls (Axios)   │ Payment Web SDKs
┌──────▼──────────┐   ┌──────▼─────────────────┐  ┌─────▼──────────────┐
│   MongoDB       │   │      Clipdrop API      │  │  Stripe / Razorpay  │
│   Collections:  │   │   (Text-to-Image v1)   │  │  Payment Gateways   │
│   user,         │   │  Returns Binary Buffer │  │  Processes Credit   │
│   transaction   │   │  Converted to Base64   │  │  Transactions       │
└─────────────────┘   └────────────────────────┘  └─────────────────────┘
```

**Decoupled Multi-Tier Architecture:**
1. **Presentation Layer (Frontend):** React Single Page Application (SPA) styled with Tailwind CSS and animated using Framer Motion. 
2. **Application Layer (Backend):** Node.js and Express.js RESTful API serving as middleware security, route handlers, and transaction coordinators.
3. **Database Layer (MongoDB):** Managed database storing User account records and Transaction metadata via Mongoose ODM.
4. **Third-Party Services Integration:** Direct server-to-server integration with Clipdrop AI API for image synthesis, and Stripe and Razorpay API instances for global and domestic payment processing.

---

## 3. Tech Stack & Justification

| Layer | Technology | Why Chosen / Justification |
| :--- | :--- | :--- |
| **Frontend** | React (Vite) | Single Page Application framework with high rendering speeds. It updates UI components efficiently via Virtual DOM diffs and simplifies state management using Hooks. |
| **Styling** | Tailwind CSS & Framer Motion | Utility-first styling avoids bulky CSS stylesheets. Framer Motion provides production-grade declarative layout animations, creating a premium interface. |
| **State Mgmt** | React Context API | Provides a global state container (`AppContext`) that feeds JWT tokens, user info, and loading states down the component tree without prop drilling. |
| **Backend** | Express.js (Node.js) | Lightweight, unopinionated framework. Perfect for designing REST APIs with custom middleware handling cross-origin requests, parsing JSON, and verifying auth states. |
| **Database** | MongoDB & Mongoose | Schemaless document store. Perfect for mapping JavaScript objects directly to database items. Mongoose provides schema validation, type checking, and query helpers. |
| **Authentication** | JWT (JSON Web Tokens) | Stateless token-based authentication. Speeds up request processing because the server does not need to look up session states in the DB for every single API call; it simply decrypts the cryptographically signed token. |
| **Security** | Bcrypt | Hashes passwords with a workload factor of 10 (`saltRounds`) using a cryptographic design resistant to brute-force rainbow table attacks. |
| **Image Generation** | Clipdrop API | External AI model engine. We call it securely from the server to abstract processing weights and protect our secret API keys from frontend codebase updates. |
| **Payments** | Stripe & Razorpay | A hybrid integration supporting both international (Stripe) and domestic (Razorpay) payment channels. |

---

## 4. Database Design (MongoDB — 2 Main Collections)

### 4.1 User Collection (`users`)
* **Purpose:** Stores customer accounts, password credentials, and active credit ledger.
* **Schema Definition:**
```javascript
const userSchema = new mongoose.Schema({
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    creditBalance: { type: Number, default: 5 },
})
```
* **Sample Document:**
```json
{
  "_id": "60d5ec49f123456789abcdef",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "password": "$2b$10$7R9jQ.678abchashedpassword...",
  "creditBalance": 4,
  "__v": 0
}
```

### 4.2 Transaction Collection (`transactions`)
* **Purpose:** Stores payment logs. Essential for reconciliation, auditing payments, and adding credits.
* **Schema Definition:**
```javascript
const transactionSchema = new mongoose.Schema({
    userId: { type: String, required: true },
    plan: { type: String, required: true },
    amount: { type: Number, required: true },
    credits: { type: Number, required: true },
    payment: { type: Boolean, default: false },
    date: { type: Number },
})
```
* **Sample Document:**
```json
{
  "_id": "60d5ec79f987654321fedcba",
  "userId": "60d5ec49f123456789abcdef",
  "plan": "Advanced",
  "amount": 50,
  "credits": 500,
  "payment": true,
  "date": 1783083419000,
  "__v": 0
}
```

---

## 5. Module-by-Module Breakdown

### Module 1: Stateless Authentication & JWT Validation
* **Registration / Login (`UserController.js`):**
  * During signup, the server hashes the user's password using `bcrypt.hash()` and saves the document. It signs a JWT token using the `jwt.sign()` API (encrypting the user's `_id` into the payload) and returns it.
  * During login, the server retrieves the user by email, compares the input password to the database hash using `bcrypt.compare()`, and returns a new JWT.
* **Auth Middleware (`auth.js`):**
  * Protected requests pass a `token` header. The custom `authUser` middleware intercepts the headers, calls `jwt.verify(token, JWT_SECRET)`, decodes the `userId`, and appends it to `req.body.userId` so subsequent controllers can automatically fetch the target user without requiring client-side inputs.

### Module 2: AI Image Generation Pipeline (`imageController.js`)
* **Core Logic:** The client submits a prompt. The endpoint runs through the auth middleware, extracting the authenticated user. The server checks the user's database record to verify that `creditBalance > 0`.
* **API Bridge:** The server instantiates a multipart `form-data` request appending the prompt, and sends a POST request using Axios to the Clipdrop API. The Clipdrop engine returns the raw generated image as a binary array buffer.
* **Data URL Packaging:** The backend converts the binary buffer into a Base64 string format (`data:image/png;base64,...`), updates the MongoDB user record by decrementing the credit balance by 1 (`creditBalance: user.creditBalance - 1`), and delivers the Base64 image payload to the client.

### Module 3: Double Payment Gateway System
* **Razorpay Workflow:**
  * When buying a credit pack, the user hits `/pay-razor`. The server initializes the Razorpay SDK and triggers `razorpayInstance.orders.create` with the package cost. The server creates an unpaid transaction entry in MongoDB and returns the Order object.
  * The frontend loads the Razorpay SDK script, opens the native Razorpay modal, collects user information, runs the payment, and calls `/verify-razor` on the backend with payment signatures.
  * The backend verifies the transaction signature. On success, it finds the transaction, toggles `payment: true`, updates the user's `creditBalance` by adding the purchased credits, and responds with a success signal.
* **Stripe Workflow:**
  * Client hits `/pay-stripe`. The backend instantiates the Stripe API and creates a checkout session using `stripeInstance.checkout.sessions.create`, defining success/cancel urls pointing to the frontend `/verify` route along with transaction ID queries.
  * The server replies with the Stripe Session URL. The React app redirects the browser directly to the Stripe Hosted Checkout page.
  * After authorization, Stripe redirects the customer back to the frontend `/verify` route. The frontend shows a loading spinner and posts validation parameters to `/verify-stripe`. The server checks the payment success parameter, flips the transaction payment flag, adds the credits to the user's account, and updates the frontend.

---

## 6. Frontend Pages & Workspaces

* **Home Page (`Home.jsx`):** Serves as the landing view. Builds brand confidence using features overview, explanatory steps, testimonial lists, and animations.
* **Image Workspace (`Result.jsx`):** The interactive core. Displays prompt text inputs, controls loading transitions (a loading bar indicating API latency), displays the rendered AI image, and provides quick download links.
* **Pricing Dashboard (`BuyCredit.jsx`):** Renders available credit tiers (Basic: 100 credits/₹10, Advanced: 500 credits/₹50, Business: 5000 credits/₹250). Initiates Stripe checkout redirections and Razorpay checkout overlays.
* **Verification Loader (`Verify.jsx`):** Intercepts search queries from payment redirects, invokes Stripe validation requests, and forwards the user back home.
* **Auth Component (`Login.jsx`):** Pop-up overlay modal switching between Signup and Login states. Validates field completeness before calling endpoints.

---

## 7. Operational Data Flow Diagrams

### AI Image Generation Flow
```
User (React View)       React Context             Auth Middleware            Image Controller            Clipdrop API
     │                         │                         │                          │                         │
     │── Writes Prompt & Submit ──>                      │                          │                         │
     │                         │── Calls axios.post ────>│                          │                         │
     │                         │   (Header: token)       │── Decrypts Token ───────>│                         │
     │                         │                         │   (Appends req.userId)   │── Checks user credits   │
     │                         │                         │                          │── Sends prompt to API ─>│
     │                         │                         │                          │                         │── Generates
     │                         │                         │                          │<── Returns Image Buffer ─│
     │                         │                         │                          │                         │
     │                         │                         │                          │── Converts to Base64    │
     │                         │                         │                          │── Decrements DB Credit  │
     │                         │<───────────────────────────────────────────────────│   (creditBalance - 1)   │
     │<── Renders Image ───────│   (Returns base64 data)
```

### Stripe Payment Flow
```
Browser             React Context             Stripe Controller            Stripe Checkout            Verify Page
   │                      │                           │                           │                       │
   │── Click Purchase ───>│                           │                           │                       │
   │                      │── POST /pay-stripe ──────>│                           │                       │
   │                      │                           │── Create db transaction   │                       │
   │                      │                           │── Init checkout session ─>│                       │
   │                      │<── Returns session.url ───│                           │                       │
   │<── Redirects User ──────────────────────────────────────────────────────────>│                       │
   │                      │                           │                           │── Pays -> Redirects ─>│
   │                      │                           │                           │   (to /verify page)   │
   │                      │                           │                           │                       │── Runs useEffect
   │                      │<── POST /verify-stripe ───────────────────────────────────────────────────────│   (Verifies ID)
   │                      │── DB updates credits ────>│                           │                       │
   │<── Show Success ─────│   (Updates local state)   │                           │                       │
```

---

## 8. Security Audits & Best Practices

| Security Feature | Implementation Detail | Target Risk |
| :--- | :--- | :--- |
| **Password Hashing** | Uses **Bcrypt** with 10 salt rounds to hash plain-text passwords before writing them to the database. | Prevents exposure of raw user passwords in the event of a database compromise. |
| **Stateless Authorization** | Issues **cryptographically signed JWTs** that are verified by the server on every request. | Prevents request tampering or unauthorized API usage. |
| **API Abstraction (Proxying)** | Routes Clipdrop API requests through the backend, keeping the API key on the server. | Prevents API key theft by hiding the keys from the frontend codebase. |
| **CORS Access Controls** | Explicit configurations using the `cors` package to declare verified origins. | Prevents Cross-Origin Resource Sharing exploits and request injections from foreign domains. |
| **Environment Isolation** | Loads credentials from a local `.env` configuration file, which is excluded via `.gitignore`. | Prevents credential leaks in shared git repositories. |

---

## 9. Key Interview Q&A

### Q: What is the main difference between JWT stateless authentication and standard session authentication?
* **Answer:** Stateful session authentication stores an ID in the database or server memory for every logged-in user, requiring a database lookup on every single request. JWT authentication is stateless. The server signs a user payload into a token. On subsequent requests, the server simply verifies the token's signature using a secret key. This eliminates database lookup latency, making it highly scalable for APIs and SPAs.

### Q: Why convert binary image buffers from the AI API into Base64 strings? Why not write the files to disk on the server?
* **Answer:** Writing images to the server's local disk is problematic because modern cloud servers (like Heroku or Vercel) have ephemeral filesystems, meaning any saved files are deleted when the server sleeps or restarts. By converting the image buffer into a Base64 data URL (`data:image/png;base64,...`), we can send the raw image data directly to the client as JSON. The frontend browser can natively render this data URL inside an `<img>` tag and download it without needing any separate file host.

### Q: How do you handle database security in this application?
* **Answer:** 
  1. We never store plain text passwords; we encrypt passwords using Bcrypt.
  2. We do not expose database URIs or credentials in the code; they are loaded into environment variables via `dotenv`.
  3. We use Mongoose schemas to strictly validate data types before saving documents to MongoDB.
  4. All write endpoints that edit database values (like image generation or credits) are wrapped in our JWT authentication middleware to prevent unauthorized access.

### Q: What happens if the backend server crashes or restarts in this setup? Do logged-in users lose their sessions?
* **Answer:** No. Since we use stateless JWT authentication, the session is stored in the browser's `localStorage` as a token, and the backend has no session state to lose. When the server restarts, it can verify incoming JWTs immediately using the `JWT_SECRET` key. This allows the backend to be completely stateless and crash-resilient.

### Q: Why did you implement both Razorpay and Stripe?
* **Answer:** To support multiple payment options. Stripe is the industry standard for international payments and cards, while Razorpay is optimized for UPI, netbanking, and domestic wallets in markets like India. Integrating both showcases the application's readiness for real-world global and local markets.

---

## 10. Future Scalability Scope
1. **Cloud Object Storage Sync:** Push generated AI images to AWS S3 or Supabase Storage and store the resulting URLs in MongoDB instead of returning raw Base64 data, reducing JSON payloads.
2. **Community Image Gallery:** Add a feed where users can toggle their generated images to "Public" so other users can browse, like, and share them.
3. **Advanced Prompt Engineering Filters:** Add dropdown selectors for styles (e.g., Cinematic, Anime, 3D Render) to enhance the prompts before sending them to Clipdrop.
4. **API Rate Limiting:** Implement the `express-rate-limit` package on our generation endpoints to prevent bots from spamming prompt requests and exhausting API keys.
