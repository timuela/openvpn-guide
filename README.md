# How to install OpenVPN for selfhosted and Enable Username/Password, 2FA Authentication
## Step 1: Install OpenVPN (if you haven't)
First, get the script and make it executable:

```bash
curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
chmod +x openvpn-install.sh
```

Then run it:

```sh
sudo ./openvpn-install.sh
```
The first time you run it, you'll have to follow the assistant and answer a few questions to setup your VPN server.

When OpenVPN is installed, you can run the script again, and you will get the choice to:

- Add a client
- Remove a client
- Uninstall OpenVPN

In your home directory, you will have `.ovpn` files. These are the client configuration files. Download them from your server and connect using your favorite OpenVPN client.

## Step 2: Enable Username and Password
1. Create the User Credentials File
Let’s say you want to store credentials in /etc/openvpn/psw-file.

Each line should contain:

```makefile
username:password
```

Example:
```bash
sudo nano /etc/openvpn/psw-file
```

Add username:password as you like, for example:
```makefile
alice:alicepassword
bob:bobpassword
```

Make it readable only by root:
```bash
sudo chmod 600 /etc/openvpn/psw-file
```

Open /etc/openvpn/server.conf file and uncommon these 2 line:
```bash
user nobody  
group nogroup  
```

2. Create an Authentication Script

OpenVPN will use a script to check the username/password against the list.

Create the script:

```bash
sudo nano /etc/openvpn/auth.sh
```
Paste this:
```bash
#!/bin/bash

# Configuration
USER_FILE="/etc/openvpn/psw-file"
TOTP_SECRET_FILE="/etc/openvpn/totp-secrets"
LOG_FILE="/var/log/openvpn-auth.log"
RATE_LIMIT_FILE="/etc/openvpn/auth-rate-limit"

# Rate limiting settings
MAX_ATTEMPTS=5  # Max attempts per minute

# TOTP settings
TOTP_WINDOW=3  # Accept codes within ±3 intervals (about ±90 seconds)

# Create log directory if not exists
mkdir -p "$(dirname "$LOG_FILE")"
touch "$LOG_FILE" "$RATE_LIMIT_FILE"
chmod 600 "$LOG_FILE" "$RATE_LIMIT_FILE"

# Log function
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
    [ "$DEBUG" = true ] && echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Rate limiting function
check_rate_limit() {
    # Get client IP (extracted from common_name which is in format "client-ip-address")
    local ip=$(echo "$common_name" | awk -F'-' '{print $1}')
    local now=$(date +%s)
    local cutoff_time=$((now - 60))  # 60-second window
    
    # Create temp file with recent attempts only
    local recent_attempts=0
    if [ -f "$RATE_LIMIT_FILE" ]; then
        while IFS=: read -r attempt_ip attempt_time; do
            if [ "$attempt_ip" = "$ip" ] && [ "$attempt_time" -ge "$cutoff_time" ]; then
                recent_attempts=$((recent_attempts + 1))
            fi
        done < "$RATE_LIMIT_FILE"
    fi
    
    # Check if exceeded limit
    if [ "$recent_attempts" -ge "$MAX_ATTEMPTS" ]; then
        log "Rate limit blocked $ip - $recent_attempts attempts in last 60 seconds"
        return 1
    fi
    
    # Log this attempt
    echo "$ip:$now" >> "$RATE_LIMIT_FILE"
    
    # Clean up old entries (older than 2x our time window)
    local cleanup_time=$((now - 120))
    if [ -f "$RATE_LIMIT_FILE" ]; then
        temp_file=$(mktemp)
        while IFS=: read -r attempt_ip attempt_time; do
            if [ "$attempt_time" -ge "$cleanup_time" ]; then
                echo "$attempt_ip:$attempt_time" >> "$temp_file"
            fi
        done < "$RATE_LIMIT_FILE"
        mv "$temp_file" "$RATE_LIMIT_FILE"
    fi
    
    return 0
}

# Start logging
log "Starting authentication process"

# Check rate limit
if ! check_rate_limit; then
    log "Authentication blocked: rate limit exceeded"
    exit 1
else
    log "Authentication allow: rate limit not exceeded"
fi

# Check if credential file exists
if [ ! -f "$1" ]; then
    log "Error: Credential file $1 not found"
    exit 1
fi

# Read credentials
username=$(head -n1 "$1" 2>/dev/null)
password=$(tail -n1 "$1" 2>/dev/null)

log "Read credentials - Username: '$username', Password: [hidden]"

# Check if username exists in password file
if ! grep -q "^$username:" "$USER_FILE"; then
    log "Error: Username '$username' not found in $USER_FILE"
    exit 1
fi

# Get stored password
stored_password=$(grep "^$username:" "$USER_FILE" | cut -d: -f2)
log "Stored password for '$username': [hidden]"

# Check if we have a TOTP secret for this user
if grep -q "^$username:" "$TOTP_SECRET_FILE"; then
    totp_secret=$(grep "^$username:" "$TOTP_SECRET_FILE" | cut -d: -f2)
    log "TOTP secret found for user '$username'"
    
    # Extract password and TOTP code (last 6 digits are TOTP)
    user_password="${password:0:$((${#password}-6))}"
    totp_code="${password:$((${#password}-6))}"
    log "Extracted - Password: [hidden], TOTP Code: [hidden]"
    
    # Verify password
    if [ "$user_password" != "$stored_password" ]; then
        log "Error: Password verification failed for '$username'"
        exit 1
    fi
    
    # Verify TOTP with window
    valid_code=0

    # Generate all valid codes in window and check if input matches any
    mapfile -t valid_codes < <(oathtool --totp -b "$totp_secret" -w $TOTP_WINDOW --now "$(date -u -d '-60 seconds' +'%Y-%m-%d %H:%M:%S UTC')" 2>/dev/null)

    log "Checking against ${#valid_codes[@]} valid codes in window..."

    for code in "${valid_codes[@]}"; do
        if [ "$totp_code" = "$code" ]; then
            valid_code=1
            log "TOTP code matched"
            break
        fi
    done
    
    if [ "$valid_code" -eq 1 ]; then
        log "Success: Password and TOTP verified for '$username'"
        exit 0
    else
        log "Error: TOTP verification failed for '$username'"
        exit 1
    fi
else
    # No TOTP required, just verify password
    log "No TOTP required for '$username'"
    if [ "$password" = "$stored_password" ]; then
        log "Success: Password verified for '$username'"
        exit 0
    else
        log "Error: Password verification failed for '$username'"
        exit 1
    fi
fi

```

Make the script executable:
```bash
sudo chmod +x /etc/openvpn/auth.sh
```

3. Modify the OpenVPN Server Configuration

Edit your OpenVPN server config /etc/openvpn/server.conf

Add:
```conf
# Enable username/password authentication
auth-user-pass-verify /etc/openvpn/auth.sh via-file

# Enforce client authentication
username-as-common-name

# Optional: push DNS and routes
script-security 3

# Plugin for PAM authentication
plugin /usr/lib/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn
```

Restart service:
```bash
sudo systemctl restart openvpn@server
```
4. Make changes to the .ovpn file

If you already created an .ovpn file, add this line to the file:
```bash
auth-user-pass
```
This enable username and password login

Or you can run the script again, and you will get the choice to:

- Add a client

Then add 'auth-user-pass' to the .ovpn file

## Step 3: Add Google Authenticator via libpam-google-authenticator

We’ll use PAM's validation engine (pam_google_authenticator.so) inside a custom script, per-user, tied to your existing psw-file users.

1. Install Google Authenticator PAM Module
```bash
sudo apt-get update
sudo apt-get install oathtool libpam-google-authenticator -y
```

2. Create the TOTP secret files:
```bash
sudo touch /etc/openvpn/totp-secrets
sudo chmod 600 /etc/openvpn/totp-secrets
```
4. Add TOTP secrets for users who need 2FA, remember that users who haven't setup 2FA can still login via username and password alone:
```bash
# Format: username:secretkey
user1:JBSWY3DPEHPK3PXP"
```

To generate secretkey:
```bash
google-authenticator
```

When it asks you to update your "/home/$USER/.google_authenticator" file, Pick no.

### The Problem:

The google-authenticator command generated a file with:

    The secret key (JBSWY3DPEHPK3PXP)

    Configuration directives (RATE_LIMIT, WINDOW_SIZE, etc.)

    Some TOTP codes (the numbers at the bottom)

    Our auth.sh script expects a simple username:secret format in the totp-secrets file.

So you just discard everything else, only keep the secret key:
```bash
sudo nano /etc/openvpn/totp-secrets
```

For each user, add a line with:
```bash
username:secretkey
```

Example:
```bash
user1:JBSWY3DPEHPK3PXP
```

## Important Notes:

For the user to set up their authenticator app, you'll need to either:

    Show them the QR code

    Give them the secret key to enter manually

After setting this up, users will need to enter: password+TOTPcode as their password in OpenVPN 

(e.g., if password is "mypass" and Google code is "123456", they'd enter "mypass123456").

Every step is logged to /var/log/openvpn-auth.log

This part of the auth.sh script keep 2 past codes valid for slow typers:
```bash
    mapfile -t valid_codes < <(oathtool --totp -b "$totp_secret" -w $TOTP_WINDOW --now "$(date -u -d '-60 seconds' +'%Y-%m-%d %H:%M:%S UTC')" 2>/dev/null)
```

You can adjust '-60 seconds' as you like, each code takes 30 seconds

log.png