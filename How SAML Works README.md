# How SAML Works
 
**Architecture Diagram, Flow Walkthrough & Troubleshooting Guide**  
Final IAM Project — Day 3 Portfolio Artifact
 
> Interview-ready explanation &nbsp;|&nbsp; Microsoft Entra ID &nbsp;|&nbsp; SAML 2.0  
> May 2026
 
---
 
## Table of Contents
 
1. [What Is SAML — In My Own Words](#1-what-is-saml--in-my-own-words)
2. [The Three Actors](#2-the-three-actors)
3. [SP-Initiated Flow — Step by Step](#3-sp-initiated-flow--step-by-step)
4. [IdP-Initiated Flow](#4-idp-initiated-flow)
5. [SAML Architecture — Component Reference](#5-saml-architecture--component-reference)
6. [Common SAML Issues & Troubleshooting](#6-common-saml-issues--how-to-troubleshoot-them)
   - [6.1 Signature Validation Failure](#61--signature-validation-failure)
   - [6.2 Audience Restriction Mismatch](#62--audience-restriction-mismatch)
   - [6.3 Assertion Expired (Clock Skew)](#63--assertion-expired-clock-skew)
   - [6.4 NameID Format Not Supported](#64--nameid-format-not-supported)
   - [6.5 User Not Assigned to the Application](#65--user-not-assigned-to-the-application)
   - [6.6 Redirect Loop Between SP and IdP](#66--redirect-loop-between-sp-and-idp)
   - [6.7 Troubleshooting Tools](#67-troubleshooting-tools)
7. [SAML vs. OIDC — When to Use Which](#7-saml-vs-oidc--when-to-use-which)
8. [Interview-Ready Explanation](#8-interview-ready-explanation)
---

## 1. What Is SAML — In My Own Words
 
SAML stands for **Security Assertion Markup Language**. It is a standard that lets one system (the Identity Provider) tell another system (the Service Provider) who a user is and what they are allowed to do — without the user ever sharing their password with the second system.
 
The best analogy is a **passport**. When traveling internationally, I don't re-prove my identity to every country from scratch. Instead, I show a document issued by a trusted authority (my government) that other countries agree to accept. SAML works the same way:
 
| Analogy | SAML equivalent |
|---|---|
| Passport office | Identity Provider (IdP) |
| The passport itself | SAML assertion |
| Foreign border control | Service Provider (SP) |


The mechanism that makes this secure is **digital signatures**. The IdP signs the assertion with its private key. The SP verifies that signature using the IdP's public key, which it received ahead of time during metadata exchange. If the signature is valid, the SP knows the assertion came from the trusted IdP and has not been tampered with in transit. The user's browser carries the assertion between them but cannot forge or alter it, it does not have the IdP's private key.
 
From the user's perspective, the experience is seamless. They click on an app, may see a login screen they recognize (or no login screen at all if they already have a session), and land in the application. **SAML is entirely invisible when it works correctly.**
 
---

## 2. The Three Actors
 
Every SAML transaction involves exactly three parties. Understanding what each one does — and what it does **not** do — is the key to understanding the protocol.
 
| Actor | Role and Responsibilities |
|---|---|
| **Identity Provider (IdP)** | Authenticates the user. Knows who the user is (directory identity, group memberships, attributes). Issues a digitally signed SAML assertion. Examples: Microsoft Entra ID, ADFS, Okta, PingFederate, Google Workspace. |
| **Service Provider (SP)** | The application the user wants to access. Does **not** know or store the user's password. Trusts the IdP's assertion. Verifies the digital signature. Creates a local session. Examples: Salesforce, Workday, AWS, ServiceNow, Zendesk. |
| **User agent (browser)** | The messenger. Carries all SAML messages between the IdP and SP as HTTP redirects (GET with URL parameters) or form POSTs. Never decodes or modifies the assertion. This is why SAML is called **browser-based SSO**. |
 
> **Critical design point:** The IdP and SP rarely communicate directly during a SAML flow — all messages travel through the browser. This works because the assertion is signed; the SP does not need to call the IdP to verify it. This makes SAML stateless between the two servers and **scalable to millions of sign-ins** without real-time IdP involvement per request.
 
---

## 3. SP-Initiated Flow — Step by Step
 
There are two SAML flows: **SP-initiated** (most common) and **IdP-initiated**. In SP-initiated flow, the user starts by going to the application, which then redirects them to the IdP. This is the standard enterprise SSO experience.
 
---
 
### Step 1 — User requests a protected resource
 
The user navigates to the application URL. The SP checks for an existing session cookie — there is none. The user is not authenticated and the SP must obtain an identity assertion.
 
---
 
### Step 2 — SP builds an AuthnRequest and redirects the browser
 
The SP constructs an XML `AuthnRequest` document and sends an HTTP 302 redirect to the IdP's SSO endpoint.
 
**Key AuthnRequest fields:**
 
| Field | Purpose |
|---|---|
| `ID` | Unique random identifier. The IdP references this in its response via `InResponseTo`, preventing CSRF. |
| `AssertionConsumerServiceURL` | Where the IdP should POST the assertion. The IdP validates this against its stored metadata. |
| `Issuer` | The SP's entity ID — its unique name in the federation. |
| `RelayState` | Opaque value the SP uses to remember where to redirect the user after auth. |
 
The SP Base64-encodes the request, signs it with its private key, and attaches it as a URL parameter along with the `RelayState`.
 
---
 
### Step 3 — Browser carries the AuthnRequest to the IdP
 
The browser automatically follows the HTTP 302 redirect. The `AuthnRequest` arrives at the IdP's SSO endpoint as a URL parameter (GET binding) or form POST (POST binding). The IdP verifies the SP's signature using the SP's public key from its stored metadata.
 
---
 
### Step 4 — IdP authenticates the user
 
1. IdP checks whether the user already has an active SSO session.
   - **Yes** → skips authentication entirely and immediately builds the assertion (transparent SSO).
   - **No** → presents a login form with MFA if required by policy.
2. Once authenticated, the IdP looks up the user's attributes from the directory: email, display name, group memberships, department, job title.
3. It builds a SAML assertion XML document and **signs it with its private key using RSA-SHA256**.
---
 
### Step 5 — IdP returns the signed assertion via the browser
 
The IdP cannot POST directly to the SP (cross-origin server-to-server communication is blocked by browser security). Instead, it returns an HTML page containing a **hidden auto-submitting form** that POSTs the Base64-encoded assertion to the SP's ACS URL. The browser becomes the carrier again.
 
**Key assertion fields:**
 
| Field | Purpose |
|---|---|
| `Issuer` | The IdP's entity ID — proves who issued the assertion. |
| `Subject / NameID` | The user's identity (usually email address or persistent identifier). |
| `NotOnOrAfter` | Expiry timestamp (typically 5 minutes) — prevents replay attacks. |
| `Recipient` | Must match the SP's ACS URL — prevents assertion theft and replay at a different SP. |
| `InResponseTo` | Matches the `ID` from the `AuthnRequest` — proves this is a response to a legitimate request. |
| `Audience` | The SP's entity ID — ensures the assertion is only valid for this SP. |
| `AttributeStatement` | Claims about the user: email, groups, roles, department. |
| `Signature` | RSA-SHA256 digital signature over the assertion XML. |
 
---
 
### Step 6 — SP validates the assertion and grants access
 
The SP's ACS endpoint receives the POST and runs a strict validation checklist:
 
- [ ] Verify the XML signature using the IdP's public key from stored metadata
- [ ] Confirm `Issuer` matches the expected IdP entity ID
- [ ] Confirm `Audience` matches the SP's own entity ID
- [ ] Check `NotOnOrAfter` has not passed *(with ~2-minute clock-skew tolerance)*
- [ ] Verify `InResponseTo` matches a pending `AuthnRequest` ID *(prevents unsolicited assertion injection)*
- [ ] Check `Recipient` URL matches the ACS URL
If all checks pass, the SP creates a local session, maps attributes to application roles, and redirects the user to the `RelayState` URL. **SAML is complete.**
 
---

## 4. IdP-Initiated Flow
 
In IdP-initiated flow, the user starts from the IdP portal (e.g., `myapps.microsoft.com`) rather than the application. They click an app tile and the IdP immediately builds and sends an assertion without waiting for an `AuthnRequest`.
 
| Aspect | Details |
|---|---|
| **No AuthnRequest** | The assertion has no `InResponseTo` field — it is unsolicited. |
| **SP must accept unsolicited assertions** | The SP must be configured to allow IdP-initiated flow; some SPs disable it for security. |
| **RelayState** | The IdP may include a `RelayState` pointing to a specific resource in the SP. |
| **Security consideration** | Without `InResponseTo`, the SP cannot verify the assertion was triggered by a legitimate user action. This is a weaker security posture. Some compliance frameworks require SP-initiated only. |
 
---
 
## 5. SAML Architecture — Component Reference
 
The table below maps each SAML component to its role in the trust chain and its real-world implementation in Microsoft Entra ID.
 
| Component | Role in Architecture / Entra ID Implementation |
|---|---|
| **Entity ID** | A URI that uniquely identifies an IdP or SP. In Entra ID, the IdP entity ID is `https://sts.windows.net/{tenant-id}/`. The SP entity ID is configured in the enterprise application registration. |
| **SAML metadata** | An XML document each party publishes containing: entity ID, SSO endpoint URL, ACS URL, and the X.509 public certificate. In Entra ID: **Enterprise Applications → SAML setup → Federation Metadata XML**. |
| **SSO endpoint (IdP)** | The URL where the IdP receives `AuthnRequest`s. In Entra ID: `https://login.microsoftonline.com/{tenant-id}/saml2` |
| **ACS endpoint (SP)** | Assertion Consumer Service — the URL on the SP where the IdP POSTs the assertion. Configured in the SP and registered in the enterprise app in Entra ID. |
| **X.509 signing certificate** | The IdP's public key certificate. The SP uses this to verify assertion signatures. In Entra ID: **Enterprise Applications → SAML-based Sign-on → SAML Signing Certificate**. |
| **NameID format** | How the user's identity is represented in the assertion. Common values: `emailAddress` (`user@domain.com`), `persistent` (opaque ID that survives email changes), `transient` (one-time ID for privacy). |
| **Attribute mapping** | Rules that control which directory attributes are included in the assertion and under what claim names. Configured in Entra ID under **User Attributes & Claims**. |
 
> **Screenshot:** Enterprise application SAML setup page in Entra ID showing Entity ID, ACS URL, and certificate
<img width="747" height="185" alt="image" src="https://github.com/user-attachments/assets/7b926c97-c5e9-4324-9228-cc49745e3400" />

---

## 6. Common SAML Issues & How to Troubleshoot Them
 
When SAML breaks, the error message is almost always shown in the browser. Use **F12 → Network tab** to capture the `SAMLResponse` POST and decode the Base64 payload to read the raw XML. Entra ID also provides sign-in logs with a Conditional Access tab and SAML-specific error codes.
 
---
 
### 6.1 — Signature Validation Failure
 
**Error:** `The SAML response signature was invalid` or `AADSTS75005`
 
| Cause | Resolution |
|---|---|
| SP has an outdated IdP certificate | Download the current **Federation Metadata XML** from Entra ID and re-import it into the SP. Entra ID rotates signing certificates periodically. |
| Certificate expiry | Check the SAML signing certificate expiry in **Entra ID → Enterprise Apps → [app] → SAML signing certificate**. Renew before expiry — Entra ID sends email notifications 60 days prior. |
| SP is validating the wrong certificate | Verify the SP is using the thumbprint that matches the current active Entra ID signing cert, not a previously imported one. |
| Assertion modified in transit | Ensure no proxy or load balancer is decoding and re-encoding the `SAMLResponse`. The assertion must arrive byte-for-byte as signed. |
 
---
 
### 6.2 — Audience Restriction Mismatch
 
**Error:** `The SAML assertion audience does not match the expected audience` or `AADSTS650056`
 
| Cause | Resolution |
|---|---|
| SP entity ID mismatch | The `Audience` in the assertion must **exactly** match the SP's entity ID. Compare the **Identifier (Entity ID)** field in Entra ID with what the SP expects — values are case-sensitive. |
| SP expecting a different format | Some SPs expect a URL; others expect a URN. Ensure the format matches the SP's documentation. |
 
---
 
### 6.3 — Assertion Expired (Clock Skew)
 
**Error:** `The token is not yet valid` or `The assertion has expired`
 
| Cause | Resolution |
|---|---|
| Server clocks out of sync | SAML assertions have a short validity window (~5 minutes). If the SP's clock is more than 2–3 minutes off from the IdP's clock, the `NotOnOrAfter` check fails. Ensure all servers are synchronized with an NTP source. |
| Assertion cache / replay | Some SP implementations cache assertions. If the cache is serving a 6-minute-old assertion, it will fail. Clear the SP session cache. |
 
---
 
### 6.4 — NameID Format Not Supported
 
**Error:** `The NameIdentifier format is not supported` or user cannot be mapped
 
| Cause | Resolution |
|---|---|
| SP expects `emailAddress`; IdP sends `persistent` | In **Entra ID → Enterprise Apps → [app] → Single sign-on → User Attributes & Claims**, change the NameID format to match what the SP expects. Most SPs want `emailAddress`. |
| User has no value for the mapped attribute | If NameID is mapped to `mail` and the user has no `mail` attribute, the assertion will contain an empty NameID. Ensure the attribute is populated in the directory for all users. |
 
---
 
### 6.5 — User Not Assigned to the Application
 
**Error:** `AADSTS50105 — The signed-in user is not assigned to a role for the application`
 
| Cause | Resolution |
|---|---|
| Assignment required is enabled but user is not assigned | In **Entra ID → Enterprise Apps → [app] → Properties**, if `Assignment required = Yes`, every user must be explicitly assigned. Go to **Users and groups** and add the user or their group. |
| User is in a group that was removed from the app | Re-add the group. Check if a sync or automation removed the assignment. |
 
---
 
### 6.6 — Redirect Loop Between SP and IdP
 
**Symptom:** The browser spins between the SP and IdP indefinitely with no error message.
 
| Cause | Resolution |
|---|---|
| SP failing assertion validation silently | Check the SP application logs. The SP may be rejecting the assertion and redirecting back to the IdP as if the user is unauthenticated. |
| ACS URL misconfiguration | The SP's ACS URL in Entra ID does not match the URL the SP is listening on — the assertion arrives at the wrong endpoint or is rejected. |
| Cookie blocking | The SP's session cookie is being blocked (e.g., `SameSite=Strict` policy in modern browsers). Check browser console for cookie warnings. |
| RelayState conflict | Some SPs reject certain `RelayState` values. Try accessing the app directly from the IdP portal (IdP-initiated) to bypass `RelayState`. |
 
---

### 6.7 Troubleshooting Tools
 
**Browser developer tools (F12 → Network tab)**
Capture the `SAMLResponse` form POST. Copy the `SAMLResponse` value and decode it from Base64 at [base64decode.org](https://www.base64decode.org/) or using PowerShell:
 
```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('<paste-value-here>'))
```
 
**SAML-tracer browser extension** *(Firefox/Chrome)*  
Intercepts and decodes SAML messages in real time without manual Base64 decoding.
 
**Microsoft Entra sign-in logs**  
Navigate to **Entra ID → Monitoring → Sign-in logs** — filter by application name and date. The **Additional details** tab shows SAML-specific error codes.
 
**Entra ID SAML toolkit**  
Available in **Enterprise Apps → [app] → Single sign-on setup page** — provides a test sign-in flow with step-by-step validation details.
 
> **Screenshot:** Entra ID sign-in log — Additional details tab showing SAML error code and description
 
---
 
## 7. SAML vs. OIDC — When to Use Which
 
SAML and OpenID Connect (OIDC) are both SSO protocols. In modern IAM you need to know both and be able to explain the difference.
 
| Dimension | SAML 2.0 | OpenID Connect / OAuth 2.0 |
|---|---|---|
| **Token format** | XML assertion — verbose, self-contained | JSON Web Token (JWT) — compact, URL-safe |
| **Transport** | Browser redirects and form POSTs | Direct API calls using HTTP + JSON; browser used only for auth code flow |
| **Best for** | Enterprise apps that predate 2012 (Salesforce, Workday, legacy SaaS) | Modern web apps, mobile apps, APIs, single-page applications |
| **Mobile support** | Poor — form POST model doesn't work well in mobile apps | Excellent — designed for mobile and API scenarios |
| **Provisioning** | No built-in provisioning | Pairs with SCIM for user provisioning |
| **Microsoft Entra default** | Used for legacy apps or when the SP only supports SAML | Preferred for new enterprise app integrations when available |
 
---
 
## 8. Interview-Ready Explanation
 
If asked **"How does SAML work?"** in an interview, here is a concise, structured answer:
 
---
 
> **On the flow:**
> "SAML is a browser-based SSO protocol with three actors: the Identity Provider, the Service Provider, and the user's browser. When a user tries to access an application, the SP redirects them to the IdP with an `AuthnRequest`. The IdP authenticates the user and issues a signed XML assertion — a document that says 'this user is alice@contoso.com, she belongs to the Sales group, and I, the trusted IdP, vouch for her.' The browser carries this assertion back to the SP as an HTTP POST. The SP verifies the digital signature using the IdP's public key, checks the assertion hasn't expired, confirms it was intended for this specific SP, and if everything checks out, grants access and creates a local session."
 
> **On security:**
> "The key security mechanism is the digital signature. The SP never needs to call the IdP to verify the assertion in real time — the signature is proof enough. This makes SAML stateless and scalable. The user's password never leaves the IdP. The SP only ever sees the assertion."
 
> **On what breaks it:**
> "Common things that break SAML: expired or rotated signing certificates, entity ID mismatches, clock skew causing assertion expiry, and users not assigned to the application in the IdP."
 
---
 
*Document prepared as a portfolio artifact for the Final IAM Project — Day 3.*
















