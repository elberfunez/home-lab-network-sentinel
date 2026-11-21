# Home Lab Network Sentinel (v4, for a Pi that is already installed)

Build a Python tool that checks TCP ports on your own Raspberry Pi, compares them with a baseline, makes an HTTP check, and saves a CSV report.

**Time:** about 3 hours, in one or two sittings\
**People:** one beginner (driver), one software engineer new to security (partner)\
**Needs:** your Raspberry Pi (already running Raspberry Pi OS, with a screen and keyboard), a Mac, a Windows laptop, and home Wi-Fi

**Starting point:** Raspberry Pi OS is already installed on the Pi with a user named `chia`. A terminal on the Pi shows the prompt `chia@raspberrypi`. Nothing else has been customized, so this guide first checks the network and turns on SSH.

**How to read this guide:** this is a learning guide. Each part starts with a **Learning goal** (what you should understand afterward), then gives numbered steps. After each step, check the **Expected** line. If it does not match, go to the **If not** line or Section 10. Do not rush past the **Explain back** questions; they are where the learning happens.

## What you will learn

By the end you should be able to explain:

-   **Networking:** IP addresses, TCP ports, clients and servers, and what open, closed and no-response really mean.
-   **Python:** functions, dictionaries, lists, sets, loops, error handling, and the standard library.
-   **Cybersecurity:** baselines, attack surface, firewalls, and why an open port is an observation and not a verdict.
-   **Engineering habits:** validating input, separating data collection from display, and keeping evidence you can compare later.

------------------------------------------------------------------------

# 1. Rules and plan

## 1.1 Rules (follow all of them)

1.  Scan **only** the Raspberry Pi you own and set up for this lab.
2.  Never scan public IPs, work or school networks, or other people's devices.
3.  Never set up router port forwarding.
4.  Treat every scan result as an **observation**, not a conclusion. An open port is not a vulnerability.
5.  Use a unique password for the Pi. Never put it in files or screenshots.

## 1.2 Working method

For every build stage in Section 6:

1.  **Predict** the result out loud.
2.  **Type** the code (do not paste).
3.  **Run** it.
4.  **Compare** with your prediction.
5.  **Explain** the new code in your own words.

The beginner types. The engineer asks questions and does not take the keyboard. Swap roles after Stage 3.

## 1.3 Schedule

  Session   Time        Do this                                         Done when
  --------- ----------- ----------------------------------------------- -----------------------------------------
  1         0:00--0:25  Check the network, enable SSH (Sec. 3)          `ssh` works from a laptop
  1         0:25--0:45  Read the concepts (Sec. 4)                      You can answer the 5 questions
  1         0:45--1:15  Update the Pi, start the web server (Sec. 5)    A laptop browser shows the test page
  2         0:00--0:10  Confirm the lab still works                     Port 8000 loads in a browser
  2         0:10--1:05  Build the scanner (Sec. 6)                      Report file is created
  2         1:05--1:45  Investigations A--F (Sec. 8)                    You explain each result
  2         1:45--2:00  Review (Sec. 12)                                Checklist complete

If you run out of time, skip Section 9. Do not skip Section 8.

------------------------------------------------------------------------

# 2. Prepare

## 2.1 Install on both laptops

1.  Install Python 3 from <https://www.python.org/downloads/>. On Windows, tick "Add Python to PATH".
2.  Install a text editor (VS Code, IDLE, or similar).

No `pip` packages are needed. You do not need Raspberry Pi Imager because your Pi is already installed.

## 2.2 Check before you start

-   The Pi is connected to your home network (Wi-Fi or Ethernet).
-   Both laptops are on the **same regular** home Wi-Fi, not a guest network.
-   No VPN is running on either laptop.
-   On macOS, if a "Local Network" permission prompt appears, click **Allow**.

------------------------------------------------------------------------

# 3. Connect to your Pi

**Learning goal:** know how to find a computer on your network (hostname and IP address), turn on a service (SSH), and connect to another computer remotely. SSH is an encrypted command-line session on another computer over the network.

Your Pi already has Raspberry Pi OS and a user named `chia`. SSH is normally **off** on a fresh install, so you will turn it on. It matters twice: it lets you use the Pi from a laptop, and it gives the scanner a real service to find on port 22.

## 3.1 Find the Pi's IP address

1.  On the Pi, open a terminal. The prompt shows `chia@raspberrypi`.
2.  Run:

``` bash
hostname -I
```

3.  Pick the private IPv4 address (`192.168.x.x` or `10.x.x.x`). Ignore long addresses with colons (IPv6).
4.  Write it down. This guide calls it **PI_IP**.

**Expected:** at least one private IPv4 address.

**If not:** the Pi is not on your network. Click the network icon at the top right of the Pi's desktop and join your home Wi-Fi (or plug in Ethernet), then run the command again.

## 3.2 Turn on SSH

On the Pi:

``` bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Enter `chia`'s password if asked. Press `q` to leave the status screen.

**Expected:** `active (running)`.

**If not:** if you see `Unit ssh.service could not be found`, run `sudo apt update` then `sudo apt install openssh-server -y`, then repeat the two commands above.

## 3.3 Connect from a laptop

1.  On the Mac use Terminal. On Windows use PowerShell or Windows Terminal.
2.  Run:

``` bash
ssh chia@raspberrypi.local
```

3.  If asked to trust the host key, type `yes`.
4.  Enter `chia`'s password. (Nothing appears while you type. That is normal.)

**Expected:** the prompt changes to `chia@raspberrypi:~ $`. You are now typing commands on the Pi from the laptop.

**If not:**

-   *Name not found:* use the IP instead: `ssh chia@PI_IP`.
-   *Connection refused:* SSH is not running. Repeat 3.2 on the Pi.
-   *Timeout:* check both devices are on the same non-guest network, no VPN is on, and the IP is current (`hostname -I` on the Pi).

From here on, run Pi commands either in this SSH session or in a terminal on the Pi itself. Both are the same Pi. Type `exit` to leave an SSH session.

------------------------------------------------------------------------

# 4. Concepts (read, then answer the questions)

-   **Server:** provides a service. **Client:** requests it. Here the Pi is the server; your scanner and browser are clients.
-   **IP address:** identifies a device on the network (for example `192.168.1.120`).
-   **TCP port:** a numbered endpoint on that device that an application listens on. IP = building, port = door.
-   **Port numbers are conventions.** 22 is usually SSH, 80 HTTP, 443 HTTPS. A number does not prove which program is listening.

## Scanner results

  Result          Meaning
  --------------- --------------------------------------------------------------------
  `OPEN`          The connection was accepted. Something is listening.
  `CLOSED`        The connection was actively refused.
  `NO RESPONSE`   Timed out or failed. A firewall, wrong IP, packet loss, or an offline device can cause this. It does **not** prove the port is closed.

## Two views

-   **Outside view:** try to connect (your scanner). Shows what is *reachable*.
-   **Inside view:** ask the Pi what it listens on (`ss`, Section 8D). Shows what is *running*.

They can disagree. That difference is the point of Investigations D and E.

## Check yourself

Answer out loud:

1.  What is the difference between an IP address and a port?
2.  Which device is the server in this lab?
3.  What service commonly uses port 22?
4.  Does an open port prove a computer was hacked?
5.  How is a timeout different from a refused connection?

**Answers:** (1) address = device, port = endpoint on it; (2) the Pi; (3) SSH; (4) no, it is an observation; (5) a timeout is no answer (often filtering), a refusal is an explicit "no."

------------------------------------------------------------------------

# 5. Start the Pi test services

**Learning goal:** see a server in action. A program *listens* on a port, and another computer connects to it. You will create exactly the thing the scanner will later look for.

Run these on the Pi: in a terminal on the Pi itself, or in an SSH session from 3.3. When a step says "open a second terminal," open a new terminal window on the Pi (or a second SSH session).

## 5.1 Update the Pi

``` bash
sudo apt update
sudo apt upgrade -y
python3 --version
```

**Expected:** updates finish (up to 10+ minutes) and a Python 3 version prints.

## 5.2 Confirm SSH is running

``` bash
sudo systemctl status ssh
```

**Expected:** `active (running)`. Press `q` to exit.

**If not:** `sudo systemctl enable --now ssh`

## 5.3 Create the test page

``` bash
mkdir -p ~/cybersecurity-lab
cd ~/cybersecurity-lab
echo "Welcome to our cybersecurity lab!" > index.html
```

## 5.4 Confirm the Pi's IP address

You found PI_IP in 3.1. Check it has not changed:

``` bash
hostname -I
```

**Expected:** the same private IPv4 address as before. If it changed, use the new one from now on.

## 5.5 Start the web server

``` bash
cd ~/cybersecurity-lab
python3 -m http.server 8000 --bind 0.0.0.0
```

Leave this terminal open. The server runs in the foreground.

Do not put sensitive files in this folder. This server is for the lab only.

## 5.6 Test it from a laptop

1.  Open a browser on a laptop.
2.  Go to `http://PI_IP:8000` (use the real number, not the text `PI_IP`).

**Expected:** the page shows `Welcome to our cybersecurity lab!`

**If not:** confirm the server is still running, the IP is correct, both devices are on the same network, and no VPN is on.

------------------------------------------------------------------------

# 6. Build the scanner (five stages)

Work on a laptop. Use the method in 1.2 at every stage.

## 6.1 Check Python

-   Mac: `python3 --version`
-   Windows: `py --version`

**Expected:** a Python 3 version prints.

## 6.2 Create the file

1.  Create a folder named `home-lab-sentinel`.
2.  Inside it, create `sentinel.py`.
3.  Check the file is not named `sentinel.py.txt`.

------------------------------------------------------------------------

## Stage 1 (6.3): check one port

**Learning goal:** functions, `try` / `except`, and what a single TCP connection attempt can and cannot tell you.

**Why this stage:** everything else is built on one small, testable function. Get it right first.

**Predict:** what does the function return for port 22, port 8000, and port 80?

**Type** into `sentinel.py` (replace the IP in the last line with PI_IP):

``` python
import socket

# How long to wait for a connection attempt, in seconds.
TIMEOUT_SECONDS = 3


def check_port(ip_address, port):
    # OPEN        = the connection was accepted
    # CLOSED      = the target actively refused it
    # NO RESPONSE = timed out, or some other network error
    try:
        with socket.create_connection((ip_address, port), timeout=TIMEOUT_SECONDS):
            return "OPEN"
    except ConnectionRefusedError:
        return "CLOSED"
    except OSError:
        # Timeouts are a kind of OSError, so they land here too.
        return "NO RESPONSE"


# Temporary test line. Delete it in Stage 2.
print(check_port("192.168.1.120", 22))
```

**Run:** `python3 sentinel.py` (Mac) or `py sentinel.py` (Windows).

**Expected:** `OPEN`. Change the port to 8000, then 80. Expected: `OPEN`, then `CLOSED`.

**If not:** closed port shows `NO RESPONSE` on Windows only → set `TIMEOUT_SECONDS = 5` (Windows is slow to report refusals). Anything else → Section 10.

**Explain back:**

-   What are the inputs and the return value of `check_port`?
-   Why is `except ConnectionRefusedError` listed before `except OSError`?
-   What does `with` do when the block ends?

------------------------------------------------------------------------

## Stage 2 (6.4): validate input, scan a list, print a table

**Learning goal:** dictionaries, loops, lists of results, and input validation.

**Why this stage:** a scanner that accepts any address could be pointed at someone else's network. Validating first is both good engineering and part of staying inside the lab rules (1.1). Storing results as data, instead of only printing them, lets later stages reuse them.

**Predict:** what should happen if the user types `hello`? And `8.8.8.8`?

1.  Delete the temporary test line.
2.  Add `import ipaddress` at the top.
3.  Add this below `check_port`:

``` python
# These are the only ports this lab checks.
COMMON_PORTS = {
    22: "SSH",
    80: "HTTP",
    443: "HTTPS",
    8000: "Python HTTP Server",
    8080: "Alternative HTTP"
}


def get_target():
    # Ask for an IP address and make sure it is a private IPv4 address.
    target = input("Enter your Raspberry Pi private IPv4 address: ").strip()

    try:
        address = ipaddress.ip_address(target)
    except ValueError:
        print("That does not look like a valid IP address.")
        return None

    if address.version != 4 or not address.is_private:
        print("Please enter a private IPv4 address for your lab.")
        return None

    return str(address)


def scan_target(ip_address):
    # Check each port in COMMON_PORTS, print a table, return the results.
    print("\n" + "=" * 58)
    print("                 HOME LAB NETWORK SENTINEL")
    print("=" * 58)
    print(f"Target: {ip_address}\n")
    print(f"{'PORT':<8}{'SERVICE':<27}{'STATUS'}")
    print("-" * 58)

    results = []

    for port, service in COMMON_PORTS.items():
        status = check_port(ip_address, port)
        print(f"{port:<8}{service:<27}{status}")

        results.append({
            "port": port,
            "service": service,
            "status": status
        })

    return results


def main():
    print("This tool is for your own private lab Raspberry Pi only.")
    target = get_target()

    if target is None:
        return

    results = scan_target(target)


if __name__ == "__main__":
    main()
```

**Run** three times: with PI_IP, with `hello`, with `8.8.8.8`.

**Expected:**

-   PI_IP: a table with 22 and 8000 `OPEN`, the rest `CLOSED`.
-   `hello`: "That does not look like a valid IP address." and no network activity.
-   `8.8.8.8`: "Please enter a private IPv4 address for your lab." and no network activity.

**Explain back:**

-   What are the keys and values in `COMMON_PORTS`?
-   What does the `for` loop repeat, and how many times?
-   Why store results in a list of dictionaries instead of only printing them?
-   Why validate input before touching the network?

------------------------------------------------------------------------

## Stage 3 (6.5): compare with a baseline

**Learning goal:** sets, and the security idea of a baseline: a description of what you *expect* a system to look like.

**Why this stage:** a list of open ports is just data. Comparing it with expectations is what turns it into something you can act on.

**Predict:** with the baseline `{22, 8000}`, what is reported if the web server is stopped?

1.  Add this constant below `COMMON_PORTS`:

``` python
# Ports we expect to be open on our lab Pi.
# Change this only if you deliberately change the lab.
EXPECTED_OPEN_PORTS = {22, 8000}
```

2.  Add this function below `scan_target`:

``` python
def compare_with_baseline(results):
    # Compare observed open ports with the ports we expected.
    observed_open_ports = {
        result["port"]
        for result in results
        if result["status"] == "OPEN"
    }

    unexpected_open = observed_open_ports - EXPECTED_OPEN_PORTS
    expected_not_observed = EXPECTED_OPEN_PORTS - observed_open_ports

    print("\nBASELINE CHECK")
    print("-" * 58)
    print(f"Expected open ports: {sorted(EXPECTED_OPEN_PORTS)}")
    print(f"Observed open ports: {sorted(observed_open_ports)}")

    if unexpected_open:
        print(f"INVESTIGATE: Unexpected open ports: {sorted(unexpected_open)}")
    else:
        print("No unexpected open ports were observed.")

    if expected_not_observed:
        print(
            "CHECK: Expected ports not observed open: "
            f"{sorted(expected_not_observed)}"
        )
    else:
        print("All expected ports were observed open.")

    return unexpected_open, expected_not_observed
```

3.  In `main()`, add after `results = scan_target(target)`:

``` python
    unexpected_open, expected_not_observed = compare_with_baseline(results)
```

**Run.**

**Expected:**

``` text
Expected open ports: [22, 8000]
Observed open ports: [22, 8000]
No unexpected open ports were observed.
All expected ports were observed open.
```

**Explain back:**

-   What does `observed_open_ports - EXPECTED_OPEN_PORTS` produce? What about the reverse?
-   Why does an unexpected port mean "investigate" and not "attack"?

**Know this:** the baseline only judges the five ports in `COMMON_PORTS`. A service on port 9999 would not be seen. "No unexpected open ports" means none *among the ports checked*.

**Swap keyboard roles now.**

------------------------------------------------------------------------

## Stage 4 (6.6): HTTP check

**Learning goal:** the difference between the transport layer (can a TCP connection be made?) and the application layer (does the program behind the port respond correctly?).

**Why this stage:** an open port only shows that *something* accepted a connection. The HTTP check asks whether it behaves like the web server you expect.

**Predict:** what status code do you expect from the Pi's page? What should happen if port 8000 is not open?

1.  Add at the top:

``` python
from urllib.error import URLError
from urllib.request import urlopen
```

2.  Add this function below `compare_with_baseline`:

``` python
def check_http(ip_address, results):
    # If port 8000 is open, request its home page and show the HTTP status.
    port_8000_is_open = any(
        result["port"] == 8000 and result["status"] == "OPEN"
        for result in results
    )

    print("\nHTTP CHECK")
    print("-" * 58)

    if not port_8000_is_open:
        print("Skipped: port 8000 was not observed open.")
        return "NOT CHECKED"

    url = f"http://{ip_address}:8000/"

    try:
        with urlopen(url, timeout=3) as response:
            status = response.status
            print(f"HTTP response: {status} {response.reason}")
            return f"{status} {response.reason}"

    except (URLError, OSError) as error:
        print(f"HTTP request failed: {error}")
        return "REQUEST FAILED"
```

3.  In `main()`, add:

``` python
    http_result = check_http(target, results)
```

**Run** with the web server running. Stop it (`Ctrl+C` on the Pi), run again, then restart it.

**Expected:** `HTTP response: 200 OK` while it runs. `Skipped: port 8000 was not observed open.` while stopped.

**Explain back:**

-   Transport layer vs application layer: what does each check prove?
-   Does `200 OK` mean the server is secure? (No. It also uses unencrypted HTTP.)

------------------------------------------------------------------------

## Stage 5 (6.7): save a timestamped CSV report

**Learning goal:** writing files, the CSV format, and keeping evidence.

**Why this stage:** a result you cannot compare later is of little use. A separate, timestamped file per scan lets you see exactly what changed between runs in Section 8.

**Predict:** if every scan wrote to the same filename, what would you lose?

1.  Add at the top:

``` python
import csv
from datetime import datetime
```

2.  Add this function below `check_http`:

``` python
def save_report(ip_address, results, unexpected_open,
                expected_not_observed, http_result):
    # Save the scan observations in a CSV file named with the scan time.
    now = datetime.now().astimezone()
    scan_time = now.isoformat(timespec="seconds")
    filename = f"security_report_{now.strftime('%Y-%m-%d_%H%M%S')}.csv"

    with open(filename, "w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)

        writer.writerow([
            "Scan time",
            "Target IP",
            "Port",
            "Service label",
            "TCP status",
            "Baseline result",
            "HTTP check"
        ])

        for result in results:
            port = result["port"]

            if port in unexpected_open:
                baseline_result = "Unexpected open port - investigate"
            elif port in expected_not_observed:
                baseline_result = "Expected open port not observed - check"
            elif result["status"] == "OPEN":
                baseline_result = "Expected open port"
            else:
                baseline_result = "Not expected to be open"

            writer.writerow([
                scan_time,
                ip_address,
                port,
                result["service"],
                result["status"],
                baseline_result,
                http_result if port == 8000 else ""
            ])

    print(f"\nReport saved: {filename}")
```

3.  Finish `main()` with:

``` python
    save_report(
        target,
        results,
        unexpected_open,
        expected_not_observed,
        http_result
    )

    print("\nScan complete. Review any items marked INVESTIGATE or CHECK.")
    print("A finding is an observation, not proof of a security problem.")
```

**Run.**

**Expected:** `Report saved: security_report_<date>_<time>.csv`, and the file exists in the folder you ran from. Open it in Excel, Numbers, or a text editor.

**If not:** compare your file with Appendix A. If the file is missing, check the folder you ran the script from.

**Explain back:**

-   Why is CSV a good format for scan results?
-   What decides the text in the "Baseline result" column?
-   Why can the report never prove the Pi is secure?
-   Why must a report never contain passwords?

------------------------------------------------------------------------

# 7. Run the finished scanner

Before each run, check:

-   [ ] The Pi is on.
-   [ ] SSH is running.
-   [ ] The web server is running on port 8000.
-   [ ] The laptop is on the same home network.
-   [ ] You know the Pi's current IP (`hostname -I`).

Run:

-   Mac: `cd ~/home-lab-sentinel` then `python3 sentinel.py`
-   Windows: `cd $HOME\home-lab-sentinel` then `py sentinel.py`

Enter PI_IP when asked.

**Expected output** (yours will differ):

``` text
==========================================================
                 HOME LAB NETWORK SENTINEL
==========================================================
Target: 192.168.1.120

PORT    SERVICE                    STATUS
----------------------------------------------------------
22      SSH                        OPEN
80      HTTP                       CLOSED
443     HTTPS                      CLOSED
8000    Python HTTP Server         OPEN
8080    Alternative HTTP           CLOSED

BASELINE CHECK
----------------------------------------------------------
Expected open ports: [22, 8000]
Observed open ports: [22, 8000]
No unexpected open ports were observed.
All expected ports were observed open.

HTTP CHECK
----------------------------------------------------------
HTTP response: 200 OK

Report saved: security_report_2026-11-21_145700.csv

Scan complete. Review any items marked INVESTIGATE or CHECK.
A finding is an observation, not proof of a security problem.
```

------------------------------------------------------------------------

# 8. Investigations

**Learning goal:** this is where the project becomes cybersecurity and not only coding. You change the lab on purpose, predict the effect, and interpret what the scanner does and does not show. Expect to learn that the scanner can be incomplete or misled, and that is the lesson.

For each one: **predict, change the lab, run the scanner, compare, explain.** Every scan saves its own report; compare the files afterward.

## 8A. Stop the web server

1.  **Predict** the result for port 8000 and the HTTP check.
2.  On the Pi, press `Ctrl+C` in the terminal running the server.
3.  Run the scanner.

**Expected:** port 8000 `CLOSED`; `CHECK: Expected ports not observed open: [8000]`; HTTP check skipped.

**Explain:** the scanner saw a service become unavailable. It did not find a vulnerability.

## 8B. Restart the web server

1.  On the Pi:

``` bash
cd ~/cybersecurity-lab
python3 -m http.server 8000 --bind 0.0.0.0
```

2.  Run the scanner.
3.  Open the reports from 8A and 8B side by side. Find the row that differs.

**Expected:** port 8000 `OPEN`, `200 OK`.

## 8C. An unexpected service appears

1.  Open a **second** terminal window on the Pi (or a second SSH session).
2.  In it, run:

``` bash
cd ~/cybersecurity-lab
python3 -m http.server 8080 --bind 0.0.0.0
```

3.  **Predict** what the baseline check will say.
4.  Run the scanner.

**Expected:** port 8080 `OPEN` and `INVESTIGATE: Unexpected open ports: [8080]`.

**Explain:** what questions you would ask if you did not start that server (what process, who started it, should it be reachable?). Then stop it with `Ctrl+C`.

## 8D. Compare outside view and inside view

1.  On the Pi, in a terminal that is not running a server, run:

``` bash
sudo ss -tlnp
```

(`-t` TCP, `-l` listening, `-n` numbers, `-p` process.)

2.  Find port 22 (`sshd`) and port 8000 (`python3`).
3.  Check the **Local Address** column: `0.0.0.0` means reachable from the network; `127.0.0.1` means this machine only.

**Explain:**

-   Where could this list and the scanner disagree?
-   Which view can see ports the scanner does not check?
-   Which view shows which *program* is listening?

## 8E. Add a firewall and watch the result change

**Order matters.** Allow SSH *before* enabling the firewall or you can lock yourself out.

1.  On the Pi:

``` bash
sudo apt install ufw -y
sudo ufw allow 22/tcp
sudo ufw allow 8000/tcp
sudo ufw enable
sudo ufw status
```

2.  Start the 8080 server again (second terminal, as in 8C).
3.  **Predict** what the scanner reports for 8080.
4.  Run the scanner.

**Expected:** port 8080 `NO RESPONSE`, even though a server is running there.

**Explain:**

-   The firewall silently drops the connection, so the scanner times out. It is not `OPEN`, and not `CLOSED`.
-   The baseline reported nothing wrong. Compare with `sudo ss -tlnp` on the Pi. Which view shows the truth?
-   A firewall limits who can reach a service. It does not secure the service.

When finished: stop the 8080 server. To undo the firewall, run `sudo ufw disable`.

## 8F. Discuss (no commands)

Answer in your own words:

-   **SSH:** if reachable, anyone who can reach it can try to log in. What reduces that risk? (Updates, strong authentication or keys, limiting who can connect, no public exposure.)
-   **Plain HTTP test server:** why is it only suitable for this lab?
-   **Attack surface:** what is it, and how does reducing services change it?
-   **Least privilege:** what is it? Give one example.

------------------------------------------------------------------------

# 9. Challenges (optional, one change at a time)

1.  **Change the baseline.** Set `EXPECTED_OPEN_PORTS = {22}`. Run while 8000 is open. **Expected:** 8000 flagged `INVESTIGATE`. Restore `{22, 8000}` afterward.
2.  **Add a known port.** Add 8080 to the baseline only if you deliberately run a service on it. Never add a port just to silence an alert.
3.  **Improve the output.** Pick one: counts of open/closed/no-response ports, a clear message when every port is `NO RESPONSE`, a short description per service, or the SSH banner for port 22.
4.  **Compare the two laptops.** Run the scanner from both. If results differ, check: same network, VPN, laptop firewall, Windows closed-port delay (Stage 1), changed Pi IP.
5.  **SSH keys.** Generate a key with `ssh-keygen -t ed25519`, copy the public key to the Pi, and test login in a **second** session before changing anything. Guide: <https://www.raspberrypi.com/documentation/computers/remote-access.html#ssh>
6.  **Compare with `nmap`** (own lab Pi only): `nmap -p 22,80,443,8000,8080 PI_IP`. Compare its `filtered` state with `NO RESPONSE`.

## For the software engineer

-   Scan ports in parallel with `concurrent.futures.ThreadPoolExecutor`.
-   Replace `input()` with `argparse` (target, ports, output file).
-   Split the logic in `compare_with_baseline` from its printing, then write unit tests.
-   Also write JSON output.

------------------------------------------------------------------------

# 10. Troubleshooting

  Symptom                                    Do this
  ------------------------------------------ -------------------------------------------------------------------------------------------
  `python3` / `py` not recognized            Install Python 3 (Section 2.1). Mac: `python3`. Windows: `py`.
  Port 22 `CLOSED`                           On the Pi: `sudo systemctl status ssh`, then `sudo systemctl enable --now ssh`.
  Port 8000 `CLOSED`                         Restart the server: `cd ~/cybersecurity-lab` then `python3 -m http.server 8000 --bind 0.0.0.0`.
  Port 8000 `NO RESPONSE`                    Check IP, Wi-Fi network, VPN, and any firewall (`ufw` on the Pi, security software on the laptop).
  Closed ports `NO RESPONSE` on Windows      Set `TIMEOUT_SECONDS = 5`.
  Every port `NO RESPONSE`                   Scanner cannot reach the Pi. Check the Pi is on, run `hostname -I` for a changed IP, same non-guest network, no VPN, macOS Local Network permission.
  Browser cannot open the page               Use the real IP and `:8000`. Do not type the text `PI_IP`.
  Report file missing                        It is saved in the folder you ran the script from. Check for an error before the save step.
  Pi's IP changed                            Run `hostname -I` on the Pi and use the new address. (Optional: reserve the IP in your router.)
  Locked out after `ufw enable`              Use the Pi's own screen and keyboard: `sudo ufw allow 22/tcp` or `sudo ufw disable`.

------------------------------------------------------------------------

# 11. Clean up

1.  Stop every Python web server (`Ctrl+C`).
2.  If you want to, disable the firewall: `sudo ufw disable`.
3.  If you do not need SSH, run `sudo systemctl disable --now ssh`. (You can turn it back on any time from the Pi with `sudo systemctl enable --now ssh`.)
4.  Run `exit` to close each SSH session.
5.  To power off cleanly, run `sudo shutdown now`, wait for the activity light to stop, then remove power.
6.  Keep the Pi updated. Never port-forward it. Keep its password private.
7.  Keep your CSV reports. Never store secrets in them.

------------------------------------------------------------------------

# 12. Review checklist

Each person explains every item in their own words.

**Networking**

-   [ ] IP address vs port.
-   [ ] Client vs server.
-   [ ] `OPEN`, `CLOSED`, `NO RESPONSE`, and why a firewall shows as `NO RESPONSE`.
-   [ ] A port number does not guarantee which program is listening.

**Python**

-   [ ] Functions, parameters, return values.
-   [ ] Dictionaries, lists, sets.
-   [ ] Loops.
-   [ ] `try` / `except`, and why the order of `except` blocks matters.
-   [ ] Standard library vs installed packages.

**Cybersecurity**

-   [ ] An open port is not automatically a vulnerability.
-   [ ] What a baseline is, and what it cannot see.
-   [ ] Outside view vs inside view.
-   [ ] Attack surface and least privilege.
-   [ ] A scan observation does not prove a system is secure or compromised.

**Software engineering**

-   [ ] Scanning, comparing, checking, and reporting are separate functions.
-   [ ] Data is collected before it is displayed or saved.
-   [ ] Input is validated before network activity.
-   [ ] Timestamped reports allow comparison over time.

------------------------------------------------------------------------

# 13. Next projects

1.  File integrity monitor (SHA-256 hashes).
2.  Packet analysis with Wireshark on your own test server.
3.  IoT security with the Arduino Uno Q and the Pi.
4.  Persistent monitoring with SQLite and scheduled scans.

Do not build a dashboard or database until you understand the scanner and its limits.

# 14. References

-   Raspberry Pi getting started: <https://www.raspberrypi.com/documentation/computers/getting-started.html>
-   Raspberry Pi SSH: <https://www.raspberrypi.com/documentation/computers/remote-access.html#ssh>
-   Python `socket`: <https://docs.python.org/3/library/socket.html>
-   Python `ipaddress`: <https://docs.python.org/3/library/ipaddress.html>
-   Python `csv`: <https://docs.python.org/3/library/csv.html>
-   Python `urllib.request`: <https://docs.python.org/3/library/urllib.request.html>
-   Nmap: <https://nmap.org>

------------------------------------------------------------------------

# Appendix A. Complete reference program

Use only to compare with your own file, or when stuck.

``` python
import csv
import ipaddress
import socket
from datetime import datetime
from urllib.error import URLError
from urllib.request import urlopen


# How long to wait for a connection attempt, in seconds.
TIMEOUT_SECONDS = 3

# These are the only ports this lab checks.
COMMON_PORTS = {
    22: "SSH",
    80: "HTTP",
    443: "HTTPS",
    8000: "Python HTTP Server",
    8080: "Alternative HTTP"
}

# Ports we expect to be open on our lab Pi.
# Change this only if you deliberately change the lab.
EXPECTED_OPEN_PORTS = {22, 8000}


def check_port(ip_address, port):
    # OPEN        = the connection was accepted
    # CLOSED      = the target actively refused it
    # NO RESPONSE = timed out, or some other network error
    try:
        with socket.create_connection((ip_address, port), timeout=TIMEOUT_SECONDS):
            return "OPEN"
    except ConnectionRefusedError:
        return "CLOSED"
    except OSError:
        # Timeouts are a kind of OSError, so they land here too.
        return "NO RESPONSE"


def get_target():
    # Ask for an IP address and make sure it is a private IPv4 address.
    target = input("Enter your Raspberry Pi private IPv4 address: ").strip()

    try:
        address = ipaddress.ip_address(target)
    except ValueError:
        print("That does not look like a valid IP address.")
        return None

    if address.version != 4 or not address.is_private:
        print("Please enter a private IPv4 address for your lab.")
        return None

    return str(address)


def scan_target(ip_address):
    # Check each port in COMMON_PORTS, print a table, return the results.
    print("\n" + "=" * 58)
    print("                 HOME LAB NETWORK SENTINEL")
    print("=" * 58)
    print(f"Target: {ip_address}\n")
    print(f"{'PORT':<8}{'SERVICE':<27}{'STATUS'}")
    print("-" * 58)

    results = []

    for port, service in COMMON_PORTS.items():
        status = check_port(ip_address, port)
        print(f"{port:<8}{service:<27}{status}")

        results.append({
            "port": port,
            "service": service,
            "status": status
        })

    return results


def compare_with_baseline(results):
    # Compare observed open ports with the ports we expected.
    observed_open_ports = {
        result["port"]
        for result in results
        if result["status"] == "OPEN"
    }

    unexpected_open = observed_open_ports - EXPECTED_OPEN_PORTS
    expected_not_observed = EXPECTED_OPEN_PORTS - observed_open_ports

    print("\nBASELINE CHECK")
    print("-" * 58)
    print(f"Expected open ports: {sorted(EXPECTED_OPEN_PORTS)}")
    print(f"Observed open ports: {sorted(observed_open_ports)}")

    if unexpected_open:
        print(f"INVESTIGATE: Unexpected open ports: {sorted(unexpected_open)}")
    else:
        print("No unexpected open ports were observed.")

    if expected_not_observed:
        print(
            "CHECK: Expected ports not observed open: "
            f"{sorted(expected_not_observed)}"
        )
    else:
        print("All expected ports were observed open.")

    return unexpected_open, expected_not_observed


def check_http(ip_address, results):
    # If port 8000 is open, request its home page and show the HTTP status.
    port_8000_is_open = any(
        result["port"] == 8000 and result["status"] == "OPEN"
        for result in results
    )

    print("\nHTTP CHECK")
    print("-" * 58)

    if not port_8000_is_open:
        print("Skipped: port 8000 was not observed open.")
        return "NOT CHECKED"

    url = f"http://{ip_address}:8000/"

    try:
        with urlopen(url, timeout=3) as response:
            status = response.status
            print(f"HTTP response: {status} {response.reason}")
            return f"{status} {response.reason}"

    except (URLError, OSError) as error:
        print(f"HTTP request failed: {error}")
        return "REQUEST FAILED"


def save_report(ip_address, results, unexpected_open,
                expected_not_observed, http_result):
    # Save the scan observations in a CSV file named with the scan time.
    now = datetime.now().astimezone()
    scan_time = now.isoformat(timespec="seconds")
    filename = f"security_report_{now.strftime('%Y-%m-%d_%H%M%S')}.csv"

    with open(filename, "w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)

        writer.writerow([
            "Scan time",
            "Target IP",
            "Port",
            "Service label",
            "TCP status",
            "Baseline result",
            "HTTP check"
        ])

        for result in results:
            port = result["port"]

            if port in unexpected_open:
                baseline_result = "Unexpected open port - investigate"
            elif port in expected_not_observed:
                baseline_result = "Expected open port not observed - check"
            elif result["status"] == "OPEN":
                baseline_result = "Expected open port"
            else:
                baseline_result = "Not expected to be open"

            writer.writerow([
                scan_time,
                ip_address,
                port,
                result["service"],
                result["status"],
                baseline_result,
                http_result if port == 8000 else ""
            ])

    print(f"\nReport saved: {filename}")


def main():
    print("This tool is for your own private lab Raspberry Pi only.")
    target = get_target()

    if target is None:
        return

    results = scan_target(target)
    unexpected_open, expected_not_observed = compare_with_baseline(results)
    http_result = check_http(target, results)

    save_report(
        target,
        results,
        unexpected_open,
        expected_not_observed,
        http_result
    )

    print("\nScan complete. Review any items marked INVESTIGATE or CHECK.")
    print("A finding is an observation, not proof of a security problem.")


if __name__ == "__main__":
    main()
```
