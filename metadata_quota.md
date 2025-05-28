## I have a GCP vm with attached disk and this attached disk is configured as home folder . So now when my application runs it stores logs and other data to the attached disk. Does this increase the number of metadata api calls?

Good question — and the short answer is:

**No, storing logs or other data on an attached disk (even as your home directory) does *not* increase the number of Metadata Server API calls.**

---

### Why?

The [GCP Metadata Server](https://cloud.google.com/compute/docs/storing-retrieving-metadata) is only accessed **when you or a process explicitly make a request** to its internal IP (`http://169.254.169.254/`) — for example:

* To get instance metadata (e.g., instance name, zone)
* To retrieve service account tokens (e.g., for authentication)
* When using client libraries with Application Default Credentials (ADC)

---

### Your Use Case:

* Logs and data stored to a mounted disk (your home folder) use **standard file I/O**, handled by the Linux kernel and the block device layer.
* The metadata server is **not involved** in normal file operations.
* So writing to `/home` (on the attached disk) **has no impact** on metadata API call volume.

---

### What *would* increase Metadata API calls?

* Using `curl http://169.254.169.254/...` manually or in scripts
* GCP client libraries (e.g., Python `google-cloud-*`) fetching auth tokens
* Tools like `gcloud`, `gsutil`, or `terraform` doing authentication or instance introspection

---

If you want to audit metadata API calls, you can:

* Use **VPC Flow Logs** (for traffic to 169.254.169.254)
* Or install tools like `tcpdump` to monitor traffic to the metadata IP

Perfect — here's how to **detect and monitor Metadata Server API calls** from your GCP VM.

---

### **1. Watch Metadata Server Traffic with `tcpdump` (quickest way)**

Run this on your VM to monitor all HTTP calls to the Metadata Server (`169.254.169.254`):

```bash
sudo tcpdump -nn host 169.254.169.254
```

If you're expecting HTTP calls (e.g., from `curl`, `gcloud`, or client libraries), you'll see them here in real-time.

#### For just HTTP GET requests:

```bash
sudo tcpdump -A -s 0 dst host 169.254.169.254 and tcp port 80
```

---

### **2. Use `strace` to trace metadata access by a specific process**

If you suspect a specific process (e.g., your app or a service) is calling the metadata server:

```bash
sudo strace -f -e trace=network -p <PID>
```

Replace `<PID>` with the process ID.

---

### **3. Use VPC Flow Logs (for long-term visibility)**

You can enable **VPC Flow Logs** on the subnet your VM is in.

#### Steps:

1. Go to **VPC network > VPC networks**.
2. Click on your network, then your subnet.
3. Edit the subnet and **enable Flow Logs**.
4. Set destination to **Cloud Logging**.
5. Use Logs Explorer to filter:

```plaintext
resource.type="gce_subnetwork"
log_id("compute.googleapis.com/vpc_flows")
jsonPayload.connection.dest_ip="169.254.169.254"
```

This gives you historical metadata server access data — including source IP (your VM), timestamps, and volumes.

---

### **4. Analyze ADC usage (if you're using service accounts)**

If you're using GCP client libraries or `gcloud`, those often request tokens from the metadata server:

```bash
curl "http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token" -H "Metadata-Flavor: Google"
```

To minimize frequent token requests, consider **caching** tokens or **reusing clients** in your code.

---

Here’s a simple Bash script you can run on your GCP VM to **monitor and log all metadata server access (169.254.169.254)** in real-time, and optionally alert if the number of calls exceeds a threshold.

---

### **Metadata API Watchdog Script**

```bash
#!/bin/bash

# === Config ===
LOG_FILE="/var/log/metadata_watchdog.log"
ALERT_THRESHOLD=10      # Alert if more than 10 requests in the window
CHECK_INTERVAL=60       # Time window in seconds
INTERFACE="eth0"        # Change if your VM uses a different network interface

# === Initialize ===
echo "Starting metadata API monitor..."
sudo touch "$LOG_FILE"
sudo chmod 644 "$LOG_FILE"

# === Monitor loop ===
while true; do
    echo "$(date): Checking metadata server traffic..." >> "$LOG_FILE"

    COUNT=$(sudo timeout "$CHECK_INTERVAL" tcpdump -nn -i "$INTERFACE" dst host 169.254.169.254 and tcp port 80 2>/dev/null | wc -l)

    echo "$(date): $COUNT calls to metadata server in last $CHECK_INTERVAL sec" >> "$LOG_FILE"

    if (( COUNT > ALERT_THRESHOLD )); then
        echo "$(date): ALERT! High metadata usage detected: $COUNT requests" >> "$LOG_FILE"
        # Optional: Add email/Slack/webhook alert here
    fi

    sleep "$CHECK_INTERVAL"
done
```

---

### **How to use:**

1. **Save it:**

   ```bash
   nano metadata_watchdog.sh
   ```

2. **Make it executable:**

   ```bash
   chmod +x metadata_watchdog.sh
   ```

3. **Run it with sudo (for tcpdump permissions):**

   ```bash
   sudo ./metadata_watchdog.sh
   ```

---

### **Optional Alert Integrations**

You can add a line inside the `if` block to alert via:

* **Email** (using `mail` or `ssmtp`)
* **Slack** (via webhook)
* **Logging service** (Cloud Logging, Datadog, etc.)

Excellent follow-up — knowing that metadata calls are happening is good, **but identifying the exact process making those calls is key** for securing or debugging GCP workloads.

---

## **Option 1: Use `ss` or `netstat` with PID tracking (for live connections)**

```bash
sudo ss -tp dst 169.254.169.254
```

This shows:

* The connection
* The PID and process name making it

**Example output:**

```bash
ESTAB 0 0 10.128.0.5:36518 169.254.169.254:http users:(("python3",pid=1234,fd=5))
```

---

## **Option 2: Use `strace` on suspicious processes**

If you suspect a certain binary (e.g., `python`, `java`, or `node` app) is making metadata requests, you can monitor it:

```bash
sudo strace -f -e connect -p <PID> 2>&1 | grep 169.254.169.254
```

That will show attempts to open a connection to the metadata server.

If you don’t know which process, you can try scanning top-level processes one by one with a loop like:

```bash
for pid in $(pgrep -u $(whoami)); do
  sudo strace -f -e connect -p $pid 2>&1 | grep -q 169.254.169.254 && echo "PID $pid is calling metadata"
done
```

---

## **Option 3: Use `auditd` (kernel-level auditing)**

This is the most robust and secure method, ideal for production or compliance environments.

1. **Install auditd** (if not already):

   ```bash
   sudo apt install auditd -y
   ```

2. **Add a rule to track access to metadata IP**:

   ```bash
   sudo auditctl -a always,exit -F dir=/ -F arch=b64 -S connect -k metadata_watch
   ```

3. **View logs**:

   ```bash
   sudo ausearch -k metadata_watch | grep 169.254.169.254
   ```

This gives you full PID, command, timestamp, and network details.

---

## **Option 4: Use eBPF tracing (like with bpftrace or Cilium)**

Advanced option. With `bpftrace`, you can trace any process making outbound connections:

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_connect /args->addr == 0xfea9fe9/ { printf("PID %d is calling 169.254.169.254\n", pid); }'
```

This is highly effective but requires kernel support and is best for advanced users.

---

## Recommended Workflow

1. **Quick test?** Use `ss -tp` or `strace` on known processes.
2. **Don't know the source?** Use a `for` loop of `strace` + `grep`.
3. **Need audit trail or production detection?** Use `auditd`.

---

## We can see google_osconfig and google_guest_ag are the services interacting with Metadata server

what we found is **expected and normal behavior** on GCP VMs.

---

## ✅ Who are these services?

### 1. **`google_osconfig`**

This is part of the [OS Config agent](https://cloud.google.com/compute/docs/manage-os). It enables features like:

* Patch management
* OS inventory reporting
* Configurations (e.g., package installation, scripts)

It calls the **Metadata Server** to:

* Get instance info
* Fetch service account tokens
* Get policy configs

### 2. **`google_guest_agent`**

This is the **Guest Agent** for Compute Engine, responsible for:

* Reporting instance status
* Managing SSH keys (from metadata)
* Configuring hostname
* Handling instance-level operations like `shutdown-script`

It constantly polls the metadata server for:

* New SSH keys
* Hostname or startup script changes
* Other signals from the GCP control plane

---

## 🧠 Should You Be Concerned?

**Generally, no** — these are **Google-maintained agents** installed by default on most GCP OS images.

They're safe, signed, and run as systemd services:

```bash
systemctl status google-osconfig-agent
systemctl status google-guest-agent
```

---

## 🔒 Want to Reduce Metadata Access?

You can **minimize (not eliminate)** access by disabling one or both agents — **only if you're sure you don’t need them**:

### Disable OS Config agent:

```bash
sudo systemctl disable --now google-osconfig-agent
```

### Disable Guest Agent:

```bash
sudo systemctl disable --now google-guest-agent
```

**⚠️ Warning:** Disabling these may break:

* SSH key propagation
* Cloud-init script delivery
* Inventory and patch reporting

So only disable if you’ve validated your app doesn’t rely on any of those features.

---

## The app does not rely on these features but users use gcloud command and cloud os login to login to these VMs.

Thanks — that’s a **crucial detail**.

Since users are using:

* ✅ `gcloud` CLI
* ✅ Cloud OS Login (`oslogin`)

then **you should NOT disable** the `google-guest-agent` or `google-osconfig-agent`. Here's why:

---

## 🔍 Why You Need Them

### ✅ `google-guest-agent` is essential for:

* **Cloud OS Login** (via [OS Login](https://cloud.google.com/compute/docs/oslogin))

  * The agent reads SSH keys and user mappings from **instance/project metadata**
* **SSH key propagation**

  * `gcloud compute ssh` uses the metadata server + this agent to inject temporary keys
* **Hostname setting**, startup/shutdown scripts, etc.

Disabling it will **break OS Login and gcloud SSH access**.

---

### ✅ `google-osconfig-agent` is used for:

* [OS Config features](https://cloud.google.com/compute/docs/manage-os) like:

  * Patch deployment
  * Package and script execution
  * Inventory reporting

If you're not using those, this agent **can be disabled safely**, but do so only if you're sure no org-level patching/inventory policies are in place.

---

## ✅ Recommendation

Since you use **Cloud OS Login**, do **not disable the guest agent**.

### What you *can* do safely:

* **Leave `google-guest-agent` enabled**
* **Optionally disable `google-osconfig-agent`** to reduce metadata calls, if:

  * You don’t use patch schedules
  * You don’t use OS inventory reporting

```bash
sudo systemctl disable --now google-osconfig-agent
```

---

## ✅ Want to reduce metadata server traffic further?

You can:

1. **Avoid polling metadata in your own app**
2. **Ensure your libraries cache tokens**
3. **Use minimal base images** that don't include unused agents
4. **Use IAP for SSH instead of Cloud OS Login** (if practical)

   * This avoids needing metadata-injected SSH keys

---

## gcloud compute ssh --project=<project> --zone=us-east1-a --interbal-ip <vm-name> is being used
Perfect — this confirms that you’re using:

> `gcloud compute ssh --internal-ip` to access VMs via **OS Login and metadata-injected SSH keys.**

---

## 🔐 What happens behind the scenes:

When you run that `gcloud compute ssh ...` command:

1. **gcloud**:

   * Generates a temporary SSH key pair
   * Uploads the **public key** to your VM's **project or instance metadata** under your user
   * This is only valid for 5–10 minutes

2. **google-guest-agent** on the VM:

   * Polls the metadata server every few seconds
   * Reads the new SSH key from metadata
   * Updates `/home/<user>/.ssh/authorized_keys`

3. **SSH client** connects to internal IP using the private key

4. Once you're connected, the key expires after a short time, and guest agent cleans it up

---

## ✅ Key takeaway:

**`google-guest-agent` is essential for this SSH login flow.**

* If you disable it, **`gcloud compute ssh` will stop working**.
* You'll either get timeout errors or be rejected due to missing keys.

---

## 🎯 Your situation:

* **DO NOT disable `google-guest-agent`**
* You *can* consider disabling `google-osconfig-agent` if you’re not using patching or OS policy configs
* If you're worried about metadata traffic:

  * The guest agent’s polling is lightweight and optimized
  * Focus on eliminating other user apps or services hitting metadata (e.g., redundant auth token fetches)

---

### 🛠 Want advanced control?

If you need to **limit metadata API access** to your app or third-party tools while keeping OS Login working, you can:

* Use **AppArmor** or **seccomp** to restrict metadata access
* Run user apps inside containers without host metadata access
* Create **custom minimal images** without the OS Config agent

## Majority API calls are for google.cloud.oslogin.dataplane.OsLoginDataPlaneService.ListLogin

Thanks — that’s a great discovery, and it makes sense in your context.

---

## 🔍 What is `google.cloud.oslogin.dataplane.OsLoginDataPlaneService.ListLogin`?

This is the **gRPC API** call used by the **Guest Agent** to:

* Fetch the **OS Login user profile**
* Retrieve **authorized SSH keys** for users
* Ensure the correct user accounts are mapped to system users (`/etc/passwd`) and have the correct `authorized_keys`

It’s invoked **each time** a user logs in via:

```bash
gcloud compute ssh ...
```

Or when:

* The Guest Agent syncs OS Login profiles periodically (e.g., every 30–60 seconds)
* There’s a metadata or user update

---

## 📈 Why you're seeing frequent `ListLogin` calls:

Because:

* You are using **Cloud OS Login**
* Users are actively logging in with `gcloud compute ssh`
* The **Guest Agent polls periodically** to keep users' accounts in sync

Even with no SSH, it might still poll — to update keys, clean up expired logins, or verify access control.

---

## ✅ Can you reduce `ListLogin` traffic?

You have **a few options**, but all come with trade-offs:

---

### **Option 1: Disable OS Login**

> 🔥 Not recommended if you're using `gcloud compute ssh`.

This would stop all `ListLogin` calls, but you'd lose:

* Integration with IAM
* Centralized SSH key management
* gcloud-based SSH login

If you *do* disable it, you'd need to manage keys manually in instance metadata or via other means (e.g., user management tools, LDAP, etc.)

---

### **Option 2: Use [OS Login with POSIX account caching](https://cloud.google.com/compute/docs/oslogin/set-up-oslogin#caching)** (✅ Recommended)

This reduces metadata calls by **caching** user login data on the VM.

#### To enable:

1. Create or update the following metadata key **on the project or VM**:

```bash
gcloud compute project-info add-metadata \
  --metadata enable-oslogin=TRUE,enable-oslogin-cache=TRUE
```

2. Or use instance-level metadata if needed:

```bash
gcloud compute instances add-metadata <vm-name> \
  --zone=<zone> \
  --metadata enable-oslogin=TRUE,enable-oslogin-cache=TRUE
```

> This reduces `ListLogin` call frequency significantly by using a local cache (typically 5–10 minutes).

---

### **Option 3: Restrict SSH access and use IAP (optional)**

If most of your users just tunnel in to run quick commands:

* Consider **IAP for SSH**, which avoids injecting keys via metadata and doesn’t rely on guest agent or OS Login.

Downside:

* Requires extra IAM setup
* Not ideal for always-on SSH sessions

---

## ✅ Summary

| Action                | Impact on Metadata Calls           | Breaks `gcloud ssh`?                |
| --------------------- | ---------------------------------- | ----------------------------------- |
| Disable OS Login      | 🔻 Significantly                   | ✅ Yes (breaks it)                   |
| Enable OS Login cache | ✅ Significantly lowers `ListLogin` | ❌ No                                |
| Use IAP for SSH       | ✅ Avoids metadata keys             | ❌ No (if IAP is set up)             |
| Disable Guest Agent   | 🔻 Eliminates traffic              | ✅ Yes (breaks SSH login & OS Login) |

---

## How to reduce the frequency of cache refresh

Thanks for checking the logs — that behavior looks **normal** and **confirms that OS Login caching is active and working properly**.

Let’s break it down:

---

## 🔍 What those log lines mean

Example:

```
oslogin_cache_refresh[1234]: Refreshing group entry cache
oslogin_cache_refresh[1234]: Refreshing passwd
```

These come from the **`oslogin_cache_refresh` binary** (part of the Guest Agent), which is responsible for refreshing:

* The **passwd cache** (user accounts)
* The **group cache** (GID/group memberships)

### 🔁 Frequency of Refreshes

By default, the Guest Agent refreshes this cache **every 4 hours** (i.e., 6 times a day), **per VM** — that’s exactly what you’re seeing.

---

## ✅ Confirmed: You’ve successfully enabled OS Login caching

Your logs show that:

* The refresh is periodic (not on every SSH login)
* The metadata `ListLogin` calls are now tied only to refreshes, not per-login
* Your setup is healthy and **optimized for reduced metadata traffic**

---

## 🧠 Advanced (Optional): Change Cache Refresh Interval

If you want to reduce it even further (e.g., only refresh every 12 hours), you can **override the systemd timer** for `oslogin-cache-refresh`.

### Step 1: Create a systemd override

```bash
sudo systemctl edit oslogin-cache-refresh.timer
```

This opens an editor. Paste the following to change the interval to every 12 hours:

```ini
[Timer]
OnBootSec=
OnUnitActiveSec=
OnBootSec=10min
OnUnitActiveSec=12h
```

Then reload:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart oslogin-cache-refresh.timer
```

---

## 🛑 Before you slow it down:

⚠️ Remember that:

* If you revoke access to a user in IAM, it may take up to 12 hours to block them.
* So choose a cache refresh rate that balances **security** and **traffic reduction**.

---











