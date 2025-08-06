# Mininet‑WiFi Project Progression

This README outlines a step‑by‑step progression for building up a Mininet‑WiFi emulation, from basic WiFi tests through to an advanced SDN‑backed handover project.

---

## 1. CLI‑Only WiFi Basics

**Goal:** Verify your Mininet‑WiFi installation and emulated radio link.

**Command:**

```bash
sudo mn --wifi \
    --topo single,2 \
    --ssid test-ssid \
    --mode g \
    --channel 1
```
![mn Wifi](/Images/Wifi.png) 

**In‑CLI Tests:**

```bash
mininet-wifi> sta1 iwconfig
mininet-wifi> sta1 ping -c3 10.0.0.2
```

Expect 0% packet loss between sta1 and sta2.

![sta1](/Images/sta1.png)  

![sta2](/Images/sta2.png)  

---

## 2. Minimal Python Script

**Goal:** Use the Python API to build a WiFi topology.

**File:** `wifi_test.py`

**Key Snippet:**

```python
from mn_wifi.net import Mininet_wifi
from mn_wifi.cli import CLI
from mininet.log   import setLogLevel, info

net = Mininet_wifi()
net.addAccessPoint('ap1', ssid='test-ssid', mode='g', channel='1', position='50,50,0')
net.addStation('sta1', ip='10.0.0.10/24', position='10,50,0')
net.configureWifiNodes()
net.build(); net.start()
net.pingAll()
CLI(net)
net.stop()
```

**Run:**

```bash
chmod +x wifi_test.py
sudo ./wifi_test.py
```

![Wifi Pingall](/Images/Pingall.png) 


---

## 3. Multi‑AP & Multi‑STA Topology

**Goal:** Emulate a small campus with multiple APs and stations.

**Tasks:**

1. Add two APs at distinct positions.
2. Add 3–4 STAs scattered around.
3. Call `net.pingAll()` and inspect associations.

![Multiple AP](/Images/MultiAP.png)  



---

## 4. Introducing Mobility

**Goal:** Simulate a moving client (STA1) crossing between two APs and observe handovers.

**Topology Recap:**
- **AP1** at (20, 80), **AP2** at (80, 20)
- **STA1...STA4**, with **STA1** mobile, others static.

**Mobility Schedule:**
```python
net.startMobility(time=0)
net.mobility(sta1, 'start', time=2,  position='10,90,0')  # t=2s
net.mobility(sta1, 'stop',  time=32, position='90,10,0')  # t=32s
net.stopMobility(time=33)
```

### 4.1 Baseline Connectivity
- **t = 0–2 s** (before movement)
  - `net.pingAll()` reports **0% packet loss** (12/12 received).
  - All STAs communicate via their initial AP associations.

### 4.2 Early Mobility (t = 2 s)
- Immediately after STA1 begins to move:
  ```
  *** pingAll at t=2s
  sta1 → sta2    RTT ≈ 39.9 ms
  sta1 → sta3    RTT ≈ 12.8 ms
  … 0% packet loss
  ```
  - STA1 still within AP1’s coverage; no handover yet.

### 4.3 Midpoint (t = 12 s)
- As STA1 approaches the overlap zone:
  ```
  *** pingAll at t=12s
  sta1 → sta2    RTT ≈ 13.3 ms
  sta1 → sta4    RTT ≈ 23.1 ms
  … 0% packet loss
  ```
  - STA1 may see AP2’s beacon but remains on AP1 until explicit reassociation.

### 4.4 Late Mobility & Handover (t = 22 s)
- Near AP2’s domain:
  ```
  *** pingAll at t=22s
  sta1 → sta2    RTT ≈ 2.6 ms    ← now served by AP2
  sta1 → sta3    RTT ≈ 8.8 ms
  … 0% packet loss
  ```
  - Monitor thread logs the handover event (associatedTo changes).
  - Connectivity stays seamless through AP migration logic.

### 4.5 Post-Mobility (t = 32 s)
- After STA1 fully crosses:
  ```
  *** pingAll at t=32s
  sta1 → sta2    RTT ≈ 6.8 ms
  sta1 → sta3    RTT ≈ 8.1 ms
  … 0% packet loss
  ```
  - STA1 is now firmly under AP2’s coverage.



---

## 5. Traffic Generation & Metrics

**Goal:** Measure throughput and latency during handover.

**Steps:**

1. Install `iperf3` on hosts: `pip install iperf3` (or system package).
2. In script or CLI, start server on one host:

   ```bash
   h1 iperf3 -s &
   ```
3. Start client on moving STA:

   ```bash
   sta1 iperf3 -c 10.0.0.20 -t 60
   ```
4. Log throughput vs. time; correlate drops with mobility schedule.

---

## 6. Hooking Up an SDN Controller

**Goal:** Use Ryu (or other) to manage flows on AP bridges.

**Commands:**

```bash
# In one terminal (venv activated):
ryu-manager --ofp-tcp-listen-port 6633 ryu.app.simple_switch_13 &
```

**Topology Mod:**

```python
from mininet.node import RemoteController
net = Mininet_wifi(controller=RemoteController)
net.addController('c1', controller=RemoteController, ip='127.0.0.1', port=6633)
```

Verify flows via:

```bash
mininet-wifi> ap1 sh ovs-ofctl dump-flows ap1
```

---

## 7. Dynamic Flow‑Rule Updates on Handover

**Goal:** Migrate an AP from one controller to another when the STA hands off.

**Logic:**

```python
if sta1.params['associatedTo'] == ap2:
    ap2.cmd('ovs-vsctl set-controller ap2 tcp:127.0.0.1:6634')
```

Measure the control‑plane handoff latency.

---

## 8. Visualization & Heatmaps

**Goal:** Plot handover metrics.

**Approach:**

* Write logs (timestamp, position, throughput) to CSV.
* Use a Jupyter notebook with `matplotlib` to:

  * Plot throughput vs. time.
  * Overlay handover point.
  * Generate heatmaps of RSSI or throughput across node positions.

---

With this roadmap and the working basics, you’ll be ready to implement and refine your advanced SDN‑backed flow‑rule migration testbed. Happy emulating!
