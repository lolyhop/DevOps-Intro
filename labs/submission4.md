# Task 1

## 1.1 Boot Performance Analysis

In this section, I measure boot time, identify the slowest startup services, and check current system load after reboot.

### Commands

```bash
systemd-analyze
systemd-analyze blame
uptime
w
```

### Output

```text
Startup finished in 4.150s (kernel) + 17.918s (userspace) = 22.068s 
graphical.target reached after 17.396s in userspace
```

```text
5.486s docker.service
3.054s cloud-init.service
2.937s cloud-init-local.service
1.731s containerd.service
1.524s man-db.service
975ms dev-vda1.device
923ms cloud-config.service
876ms google-guest-agent.service
859ms systemd-udev-settle.service
846ms ModemManager.service
# ... (remaining services)
```

```text
18:37:52 up 2 min, 1 user, load average: 0.19, 0.18, 0.08
```

```text
18:37:53 up 2 min, 1 user, load average: 0.19, 0.18, 0.08
USER     TTY   FROM            LOGIN@  IDLE   JCPU   PCPU WHAT
chrnegor pts/0 93.158.188.XXX  18:36   1.00s  0.03s  0.00s w
```

### Observations

- Total boot time is about **22s**, with userspace init taking the larger part;
- `docker.service` and `cloud-init` are the biggest contributors to startup delay;
- Right after boot, load averages are low, which indicates no CPU pressure;
- Current interactive load is minimal (1 active user session).

**Note:** All of the commands have been run on a fresh Yandex Cloud VM (2 min uptime). Cloud-init services handle VM setup tasks like SSH keys and network config.

## 1.2 Process Forensics

### Commands

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -n 6
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 6
```

### Output

```text
PID  PPID CMD                         %MEM %CPU
932  1    /usr/bin/dockerd -H fd:// - 0.5  0.6
718  1    /usr/bin/containerd         0.2  0.1
788  1    /usr/bin/python3 /usr/share 0.1  0.0
697  1    /usr/bin/python3 /usr/bin/n 0.1  0.0
750  1    /usr/bin/google_guest_agent 0.1  0.0
```

```text
PID  PPID CMD                         %MEM %CPU
1    0    /sbin/init                   0.0  1.8
932  1    /usr/bin/dockerd -H fd:// -  0.5  0.5
17   2    [migration/1]                0.0  0.2
23   2    [migration/2]                0.0  0.2
29   2    [migration/3]                0.0  0.2
```

### Observations

- **Top memory-consuming process:** `dockerd` (PID 932, 0.5% MEM);
- CPU usage is also low; no runaway process is detected;
- Resource usage pattern suggests an idle VM with baseline daemon activity.

## 1.3 Service Dependencies

### Commands

```bash
systemctl list-dependencies
systemctl list-dependencies multi-user.target
```

### Output

```text
default.target
├─accounts-daemon.service
├─apport.service
├─display-manager.service
├─e2scrub_reap.service
└─multi-user.target
  ├─containerd.service
  ├─docker.service
  ├─ssh.service
  ├─systemd-networkd.service
  ├─systemd-resolved.service
  ├─ufw.service
  ├─unattended-upgrades.service
  └─...
```

```text
multi-user.target
├─containerd.service
├─docker.service
├─ssh.service
├─systemd-networkd.service
├─systemd-resolved.service
├─basic.target
│ ├─sockets.target
│ ├─sysinit.target
│ └─timers.target
├─cloud-init.target
└─getty.target
# ...
```

### Observations

- The OS starts many services through dependencies, not just one service;
- The dependency tree shows that `multi-user.target` relies on lower-level targets such as `basic.target` and `sysinit.target`;
- `containerd` and `docker` are included in normal system startup.

## 1.4 User Sessions

### Commands

```bash
who -a
last -n 5
```

### Output

```text
system boot  2026-02-21 18:35
LOGIN ttyS0  2026-02-21 18:35
LOGIN tty1   2026-02-21 18:35
run-level 5  2026-02-21 18:36
chrnegor + pts/0 2026-02-21 18:36 . 3913 (93.158.188.XXX)
```

```text
chrnegor pts/0 93.158.188.XXX Sat Feb 21 18:36   still logged in
reboot   system boot          Sat Feb 21 18:35   still running
chrnegor pts/0 93.158.188.XXX Sun Feb  1 11:25 - 15:00 (03:35)
chrnegor pts/1 93.158.188.XXX Sun Feb  1 09:10 - 11:22 (02:12)
chrnegor pts/0 93.158.188.XXX Sun Feb  1 08:31 - 11:07 (02:35)
```

### Observations

- Current system state matches a recent reboot with one active remote login;
- Historical sessions indicate regular SSH access patterns;
- IP addresses were sanitized by replacing the last octet with `XXX`.

## 1.5 Memory Analysis

### Commands

```bash
free -h
cat /proc/meminfo | grep -e MemTotal -e SwapTotal -e MemAvailable
```

### Output

```text
total used free shared buff/cache available
Mem:  15Gi 230Mi 14Gi 1.0Mi 408Mi 15Gi
Swap: 0B   0B    0B
```

```text
MemTotal:     16381328 kB
MemAvailable: 15854340 kB
SwapTotal:           0 kB
```

### Observations

- The system has 16GB total memory (15Gi displayed due to binary vs decimal conversion);
- Memory pressure is very low: most RAM is available;
- Swap is disabled (`SwapTotal: 0`), but current workload does not require swap;


## Task 1 Summary

- Total boot time is **22.068s** and the slowest units are `docker.service` (**5.486s**), `cloud-init.service` (**3.054s**), and `cloud-init-local.service` (**2.937s**);
- Runtime load is low right after login: `load average 0.19, 0.18, 0.08` with one active user session;
- Top memory-consuming process is `dockerd` (PID `932`) at about **0.5% MEM**; CPU leaders are also low (PID 1 at **1.8% CPU**, `dockerd` around **0.5% CPU**);
- Memory state confirms low pressure: `MemTotal 16381328 kB`, `MemAvailable 15854340 kB` (about **96.8%** available), and `SwapTotal 0 kB`;
- Overall, the VM is healthy and lightly loaded, with no signs of resource bottlenecks or abnormal process behavior during this snapshot.

---

# Task 2

## 2.1 Network Path Tracing


Let's check the route to `github.com` and verify DNS resolution details.

### Commands

```bash
traceroute github.com
dig github.com
```

### Output

```text
traceroute to github.com (140.82.121.3), 30 hops max, 60 byte packets
 1  * * *
 2  yandeks-ic-370832.ip.twelve99-cust.net (213.248.94.XXX)  4.033 ms  3.560 ms  3.901 ms
 3  yandeks-ic-370832.ip.twelve99-cust.net (213.248.94.XXX)  4.083 ms  4.072 ms  3.774 ms
 4  mow-b2-link.ip.twelve99.net (213.248.94.XXX)  3.603 ms  3.591 ms  3.577 ms
 5  sto-bb1-link.ip.twelve99.net (62.115.143.XXX)  23.604 ms  23.093 ms  23.457 ms
 6  ffm-bb1-link.ip.twelve99.net (62.115.143.XXX)  46.360 ms  44.847 ms  44.827 ms
 7  ffm-b11-link.ip.twelve99.net (62.115.124.XXX)  45.234 ms  45.504 ms  45.578 ms
 8  github-ic-350972.ip.twelve99-cust.net (62.115.182.XXX)  39.235 ms  39.056 ms  39.497 ms
 9  * * *
10  * * *
# ... hops 11-30 are also * * *
```

```text
; <<>> DiG 9.18.30-0ubuntu0.20.04.2-Ubuntu <<>> github.com
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 709
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; QUESTION SECTION:
;github.com.                    IN      A
;; ANSWER SECTION:
github.com.             12      IN      A       140.82.121.3
;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

### Observations

- The route passes through provider backbone nodes (Twelve99 network) and reaches a GitHub-connected edge node;
- Missing responses after hop 8 are normal in many networks because intermediate devices may filter ICMP;
- DNS is healthy and very fast in this snapshot.

## 2.2 Packet Capture 

### Command

```bash
sudo tcpdump -ni any -vvv udp port 53
```

### Notes

- Initial attempt with `timeout` returned 0 packets;
- Additional step was performed: I kept `tcpdump` running and generated fresh DNS requests (`dig @8.8.8.8 github.com` and `dig @1.1.1.1 cloudflare.com`) from another terminal;
- DNS request/response packets were captured successfully (4 packets total).

### Output

```text
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
19:43:18.610402 IP 10.128.0.XXX.59839 > 8.8.8.XXX.53: 4137+ A? github.com. (51)
19:43:18.671585 IP 8.8.8.XXX.53 > 10.128.0.XXX.59839: 4137 q: A? github.com. 1/0/1 github.com. A 140.82.121.XXX (55)
19:43:18.689926 IP 10.128.0.XXX.40959 > 1.1.1.XXX.53: 32954+ A? cloudflare.com. (55)
19:43:18.696939 IP 1.1.1.XXX.53 > 10.128.0.XXX.40959: 32954 q: A? cloudflare.com. 2/0/1 cloudflare.com. A 104.16.132.XXX, A 104.16.133.XXX (75)
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```


### DNS Query Example

```text
IP 10.128.0.XXX.59839 > 8.8.8.XXX.53: 4137+ A? github.com.
```

### Observations

- DNS traffic appears only when new queries are actively generated during capture;
- The capture shows clear request/response pairs for `github.com` and `cloudflare.com`;
- Each DNS query includes a unique transaction ID (4137, 32954) to match requests with responses.

## 2.3 Reverse DNS

### Commands

```bash
dig -x 8.8.4.4
dig -x 1.1.2.2
```

### Output

```text
; <<>> DiG 9.18.30-0ubuntu0.20.04.2-Ubuntu <<>> -x 8.8.4.4
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13377
;; QUESTION SECTION:
;4.4.8.8.in-addr.arpa.          IN      PTR
;; ANSWER SECTION:
4.4.8.8.in-addr.arpa.   39826   IN      PTR     dns.google.
;; Query time: 4 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

```text
; <<>> DiG 9.18.30-0ubuntu0.20.04.2-Ubuntu <<>> -x 1.1.2.2
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 43491
;; QUESTION SECTION:
;2.2.1.1.in-addr.arpa.          IN      PTR
;; Query time: 96 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

### Comparison

- First IP (`8.8.4.4`) has a valid PTR mapping to a provider hostname;
- Second IP (`1.1.2.2`) did not return a PTR record in this check;
- This shows reverse DNS availability differs by address block and provider configuration.

## Key Concepts Learned

- `traceroute` maps reachable network path segments and per-hop latency;
- `dig` validates forward DNS resolution and returns TTL/response metadata;
- `tcpdump` confirms whether DNS packets are present in live traffic;
- Reverse DNS (`dig -x`) may return either valid PTR records or `NXDOMAIN`, depending on provider settings.

---

## What I Learned From This Lab

- I can break down Linux boot behavior using `systemd-analyze` and identify which services slow startup the most;
- I can inspect process and memory usage with `ps`, `uptime`, and `free` to quickly judge whether a machine is under load;
- I understand how service dependencies (`systemctl list-dependencies`) explain startup order and runtime service relationships;
- I can use `traceroute`, `dig`, and reverse DNS lookups to diagnose network path and DNS behavior end-to-end.
