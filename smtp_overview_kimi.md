# SMTP Configuration Overview

## SMTP Components Explained

### **SMTP Server Address**
This is the hostname of the mail server that actually sends your emails. It's provided by your email service provider.

**Examples:**
- Gmail: `smtp.gmail.com`
- SendGrid: `smtp.sendgrid.net`
- Mailgun: `smtp.mailgun.org`
- AWS SES: `email-smtp.us-east-1.amazonaws.com`

The server listens on specific ports (usually 587 or 465) and handles the actual delivery of your emails to recipients.

**Important:** These SMTP servers run on the email provider's machines (like Gmail's servers), not your local machine. Your application connects to these remote servers via TCP sockets and uses the SMTP protocol to submit emails for delivery.

### **SMTP Authentication User**
This is the username/identity used to log into the SMTP server. It's almost always your email address.

**Examples:**
- `your-email@gmail.com`
- `noreply@your-company.com`
- `username@sendgrid.net`

It proves to the SMTP server that you have permission to send emails from that account.

### **SMTP Authentication Password**
This is the credential used alongside the username to authenticate with the SMTP server.

**Important note:** For Gmail and many providers, this is NOT your regular account password. Providers give you an "app password" or "API key" for security reasons—your real password never gets exposed to applications.

### **Sender Email Address**
This is the email that appears in the "From" field when recipients receive your email.

```
From: noreply@example.com  ← This is the sender email address
To: user@recipient.com
Subject: Password Reset
```

It doesn't have to be the same as the SMTP authentication user, but the SMTP server must authorize you to send from that address.

### **Enable TLS Encryption**
TLS (Transport Layer Security) encrypts the connection between your application and the SMTP server.

- **What it does:** Creates a secure tunnel so your password and email content can't be intercepted
- **How it works:** Connection starts unencrypted, then upgrades to encryption after authentication ("STARTTLS")
- **Port:** 587 (standard)
- **When to use:** Modern providers (Gmail, SendGrid, etc.)
- **Protocol complexity:** More than just "sending requests to a URL" - involves TCP socket connections, SMTP protocol handshakes, authentication commands, and TLS negotiation

### **Enable SSL Encryption**
SSL (Secure Sockets Layer) is the predecessor to TLS, but the terms are often used interchangeably now.

- **What it does:** Similar to TLS—encrypts the connection
- **How it works:** Connection is encrypted from the start (before authentication)
- **Port:** 465 (standard)
- **When to use:** Older servers, or providers that specifically require it
- **Protocol complexity:** Also involves multi-step SMTP protocol communication, not simple HTTP requests

### **SMTP Port**
The network port where the SMTP server listens. Different ports indicate different security protocols:

**Why you need the port:** The hostname (like `smtp.gmail.com`) gets you to the server, but the port tells it which service/protocol you want to use. Without specifying the port, your app wouldn't know where to connect on the server.

| Port | Protocol | Use Case |
|------|----------|----------|
| **25** | No encryption | Internal/legacy systems only (not recommended) |
| **587** | TLS (STARTTLS) | Modern standard, most providers |
| **465** | SSL | Older but still used by some providers |

---

## How SMTP Actually Works

Unlike simple HTTP requests, SMTP involves:

1. **TCP socket connection** to the SMTP server (not HTTP)
2. **Multi-step protocol handshake** with SMTP commands
3. **Authentication** using your credentials
4. **Message formatting** per RFC standards
5. **TLS/SSL negotiation** for encryption

Your application code handles the high-level logic, but libraries like `emails` manage the complex SMTP protocol communication under the hood.

## What Mailcatcher Does

Mailcatcher is a **fake SMTP server for local development**. Instead of actually sending emails, it:

1. **Accepts emails** from your application as if it were a real SMTP server
2. **Stores them in memory** (never leaves the machine)
3. **Shows a web UI** at `http://localhost:1080` where you can view, inspect, and debug emails
4. **Allows testing** without needing real SMTP credentials or sending real emails

**Example workflow:**
```
Your app → Mailcatcher SMTP (port 1025) → Email stored
         → View at http://localhost:1080
```

This is perfect for testing password reset emails, new account notifications, etc., without actually sending anything.

---

## Setting Up Gmail as Your SMTP Host

Gmail requires special setup because it doesn't allow direct password authentication for security reasons. Here's how:

### **Step 1: Enable 2-Factor Authentication**
1. Go to [myaccount.google.com/security](https://myaccount.google.com/security)
2. Enable 2-Step Verification (if not already enabled)

### **Step 2: Create an App Password**
1. Return to Security settings
2. Find "App passwords" (only appears if 2FA is enabled)
3. Select "Mail" and "Windows Computer" (or your device type)
4. Google generates a 16-character password
5. **Copy this password** (you'll use it as `SMTP_PASSWORD`)

### **Step 3: Configure Your `.env`**
```bash
SMTP_HOST=smtp.gmail.com
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=xxxx xxxx xxxx xxxx    # The 16-char app password (spaces optional)
EMAILS_FROM_EMAIL=your-email@gmail.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
```

### **Step 4: Test**
Run your application and use the test email endpoint or trigger an email action. The email should be sent from your Gmail account.

**Important:** Never use your actual Gmail password here. Always use the app-specific password.

---

## Does Sending Emails Cost Money?

**Short answer: It depends on the provider and volume.**

### **Free/Affordable Options:**

| Provider | Free Tier | Cost |
|----------|-----------|------|
| **Gmail** | Yes, unlimited | Free (personal account only) |
| **SendGrid** | 100 emails/day | $19.95/month for higher volume |
| **Mailgun** | 5,000 emails/month | $0.50 per 1,000 after free tier |
| **AWS SES** | 62,000 emails/month (first year) | $0.10 per 1,000 emails |
| **Brevo (Sendinblue)** | 300 emails/day | Free tier, $20+/month for more |

### **Gmail Considerations:**
- **Free for personal accounts** (no cost)
- **Subject to rate limits:** ~500 emails per day
- **Not recommended for production** (rate limits, reliability)
- **Better for:** Side projects, internal notifications, testing

### **Production Recommendation:**
For a real application, use a dedicated email service (SendGrid, Mailgun, AWS SES). They're:
- More reliable
- Handle bounce/spam complaints
- Provide better deliverability
- Scale to any volume
- Provide detailed analytics

---

## TLS vs SSL: Comparison and Recommendation

### **Side-by-Side Comparison**

| Feature | TLS | SSL |
|---------|-----|-----|
| **Age** | Modern (2008+) | Legacy (1990s) |
| **Security** | Stronger, modern cryptography | Weaker, outdated algorithms |
| **Port** | 587 | 465 |
| **Connection start** | Unencrypted → STARTTLS upgrade | Encrypted from start |
| **Speed** | Slightly more overhead (handshake) | Slightly faster initial connection |
| **Provider support** | Nearly universal | Most modern providers |
| **Vulnerability risk** | Very low | Higher (deprecated protocol) |

### **How They Differ**

**TLS (port 587) - "STARTTLS":**
```
1. Connect unencrypted to SMTP server
2. Client says "STARTTLS"
3. Server upgrades connection to encryption
4. Authentication happens over encrypted connection
```

**SSL (port 465) - "Implicit TLS":**
```
1. Connection is encrypted from the moment you connect
2. No upgrade step needed
3. Authentication happens over encrypted connection
```

### **Which to Use?**

**Use TLS (port 587)** ✅
- Modern standard
- Works with: Gmail, SendGrid, Mailgun, AWS SES, most providers
- More flexible
- Better error handling during protocol negotiation

**Use SSL (port 465)** ⚠️
- Only if provider explicitly requires it
- Check provider documentation first
- Some providers support both; TLS is preferred

### **For Your Gmail Setup:**
```bash
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
```

---

## Quick Reference: Setting Up Gmail

```bash
# .env configuration
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password    # 16 chars from Google
EMAILS_FROM_EMAIL=your-email@gmail.com
SMTP_TLS=True
SMTP_SSL=False

# Test it works:
# POST /api/v1/utils/test-email/
# with body: {"email_to": "recipient@example.com"}
```
