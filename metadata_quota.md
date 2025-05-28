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





