# 🔓 Challenge: Natas Level 21 → 22

## 📋 Objective

Find the password for **Natas 22**, which requires logging in as an administrator to retrieve the credentials.

---

## 🔍 Analysis

The application checks if you are logged in as an admin user. The main page displays:

```
You are logged in as a regular user.
```

To proceed, you need to escalate privileges to admin status.

**Key Hint:** *This website is colocated with natas21-experimenter.*

### Vulnerable Code

**Main Application:**

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

**The Experimenter (Colocated Application):**

```php
session_start();

if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}
```

**Critical Flaw:** Every request parameter is directly injected into the session without any validation or sanitization.

---

## 💡 Exploitation Strategy

The vulnerability lies in the experimenter application's unsafe handling of user input. Since both applications share the same session (they are colocated), we can exploit the experimenter to inject arbitrary session variables.

### Key Insight

The structure of the vulnerable code is:

```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```

This loop blindly copies any request parameter into the session. By sending a request with an `admin` parameter set to `1`, we can manipulate the session data directly.

---

## ✅ Solution

### Payload

Visit the experimenter application with the following parameters:

```
/index.php?submit=1&admin=1
```

### Execution Flow

**Our Request:**
```
GET /index.php?submit=1&admin=1
```

**Resulting PHP Execution:**
```php
$_SESSION["admin"] = "1";
```

### What Happens

1. The `submit` parameter triggers the vulnerable `foreach` loop
2. The `admin=1` parameter gets injected directly into `$_SESSION`
3. The session now contains `$_SESSION["admin"] = 1`
4. When you access the main Natas 21 page, it reads the manipulated session
5. The condition `if($_SESSION["admin"] == 1)` evaluates to **true**
6. The admin credentials for Natas 22 are revealed

---

## 🛡️ Prevention & Remediation

### Why Direct Session Injection Fails

This challenge demonstrates that **blindly trusting user input for session data is dangerous**. Without validation, attackers can set any session variable they want.

### Secure Alternatives

1. **Best Practice:** Use a whitelist of allowed session keys
   ```php
   $allowed_keys = ['username', 'email'];
   
   if(array_key_exists("submit", $_REQUEST)) {
       foreach($_REQUEST as $key => $val) {
           if(in_array($key, $allowed_keys)) {
               $_SESSION[$key] = $val;
           }
       }
   }
   ```

2. **Validate and Sanitize Input**
   ```php
   if(array_key_exists("submit", $_REQUEST)) {
       if(!empty($_REQUEST['admin']) && $_REQUEST['admin'] === '1') {
           $_SESSION["admin"] = 1; // Only set admin through proper logic
       }
   }
   ```

3. **Never Allow Direct Session Manipulation from User Input**
   ```php
   // Use dedicated authentication functions
   if(authenticate_user($_REQUEST['username'], $_REQUEST['password'])) {
       $_SESSION["admin"] = check_admin_status($_REQUEST['username']);
   }
   ```

---

## 🎯 Key Takeaway

- **Never blindly inject user input into session variables**
- **Whitelist allowed session keys** instead of accepting any parameter
- **Validate and sanitize all user input** before storing in sessions
- **Separate authentication logic** from direct session manipulation
- Sessions are critical security boundaries — treat them with utmost care
