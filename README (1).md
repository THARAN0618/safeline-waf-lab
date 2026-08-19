# SafeLine WAF Lab

This is a small cybersecurity lab I did to learn how a Web Application Firewall (WAF) actually works. I deployed SafeLine WAF on Kali Linux using Docker, put it in front of a test web server, and then attacked my own setup with a few common exploit payloads to see if it would actually block them.

I'm still learning, so this repo documents everything I did, including the mistakes I made along the way.

## Why I did this

I had read about WAFs before but never actually set one up myself. I wanted to understand what a WAF does in practice, not just in theory, so I decided to build a small lab and test it against real attack payloads.

## How it's set up

The idea is simple. Instead of attacking the web server directly, all traffic first goes through the WAF. The WAF checks the request and either forwards it to the server if it looks safe, or blocks it if it looks like an attack.

```
Attacker
   |
   |  request to port 80
   v
SafeLine WAF
   |
   |  clean traffic forwarded to port 8000
   v
Python test web server
```

## Tools used

- Kali Linux
- Docker and Docker Compose
- SafeLine WAF (Community Edition)
- Python's built-in http.server, used as the test target

## Steps I followed

### 1. Installed Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin -y
sudo systemctl enable --now docker
```

I checked that Docker was actually running before moving on:

```bash
sudo systemctl status docker
```

### 2. Deployed SafeLine WAF

```bash
sudo bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/setup.sh)"
```

This installed the WAF containers (management console, engine, database).

Then I reset the admin password to log in:

```bash
sudo docker exec -it safeline-mgt resetadmin
```

The dashboard is available at https://127.0.0.1:9443/

### 3. Set up a test web server

```bash
mkdir -p ~/testweb
cd ~/testweb
echo "Hello, Demo Site" > index.html
python3 -m http.server 8000
```

This server has no protection of its own on purpose, so anything that gets blocked later is because of the WAF, not the server.

### 4. Connected the WAF to the test server

In the SafeLine dashboard, under Applications, I added a new application with these settings:

- Listening Port: 80
- Upstream Protocol: HTTP
- Upstream Host: 172.17.0.1
- Upstream Port: 8000

I used 172.17.0.1 instead of 127.0.0.1 because the WAF runs inside Docker, and from inside a container, 127.0.0.1 refers to the container itself, not my actual machine. 172.17.0.1 is the Docker host gateway address, which lets the container reach services running on my host.

I also made sure the application was set to Defense mode, not Audited mode, since Audited mode only logs attacks without blocking them.

### 5. Tested it with some attack payloads

I sent these requests directly to the WAF on port 80:

| Attack type | Payload | Result |
|---|---|---|
| SQL Injection | /?id=1' OR '1'='1 | Blocked, 403 Access Forbidden |
| Cross-Site Scripting | /?q=<script>alert(1)</script> | Blocked |
| Path Traversal | /../../etc/passwd | Blocked |
| Normal request | / | Allowed |

The blocked requests also showed up in the Attacks section of the dashboard, along with the source IP, attack type, and time.

## Problems I ran into

I did not get everything right the first time, so I'm writing down what actually went wrong.

**Docker exec permission error**
I ran the resetadmin command without sudo the first time and got a permission error. I was not part of the docker group, so I just used sudo for that command instead.

**Attacks were not getting blocked at first**
My first few test payloads went to 127.0.0.1:8000 directly, which is the test server, not the WAF. So of course nothing was blocked, since I was bypassing the WAF completely. Once I sent the payloads to port 80 instead, they got blocked properly.

**Confused about upstream host**
I originally tried 127.0.0.1 as the upstream host in the WAF settings, thinking it would point back to my own machine. It did not work because the WAF container sees 127.0.0.1 as itself, not my host machine. I had to use the Docker gateway IP 172.17.0.1 instead.

**Thought the WAF was not working, but it was just in the wrong mode**
For a while I thought the attacks were not being blocked properly. It turned out the application was set to Audited mode, which only records attacks without blocking them. Switching to Defense mode fixed this.

## What I learned

- A WAF sits in front of an application and filters traffic without needing any changes to the app itself.
- Getting the networking right (which IP and port to use) was actually harder than configuring the WAF itself.
- Testing in Audited mode first is safer before switching to full blocking, since it lets you check what would be blocked without actually breaking anything.
- The dashboard logs are useful for understanding what the WAF is actually detecting, not just whether a request was blocked or not.

## Screenshots

The screenshots folder in this repo contains images from each step, including the Docker setup, the WAF dashboard, the reverse proxy configuration, and the blocked attack attempts with their logs.

## Notes

This is a personal learning project, not a production setup. If anyone has suggestions for what else I should test, feel free to open an issue.
