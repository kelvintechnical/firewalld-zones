# Lab: Exploring firewalld Zones — Catalog and Compare Trust Models

- **Series:** linux-ops-mastery — RHCSA Firewall
- **Subjects covered:** Zone abstraction, `firewall-cmd --get-zones`, `--get-default-zone`, `--list-all-zones`, `--info-zone=ZONE`, trust defaults (`public`, `home`, `internal`, `trusted`, `drop`), runtime vs permanent vocabulary (introduced; persistence in later labs)
- **Career arcs covered:** RHCSA (zone selection is a common EX200 theme), RHCE (Ansible `firewalld` zone parameters), SRE (multi-homed trust boundaries), DevOps (golden images ship `public` by default), AI/MLOps (segmenting admin vs data planes)
- **Prerequisite:** `systemctl` basics; this lab is conceptual cataloging — not yet changing defaults
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 service health · 2–3 enumerate and compare zones · 4 deep `info-zone` inspection · 5 edge: services vs ports lists · 6 capstone matrix + cleanup

---

## Objective

`firewalld` hides raw `iptables`/`nft` behind **zones**: named bundles of allowed services, ports, masquerading, and forwarding policy. This lab is the **tourist map** — you will walk every shipped zone, read its description, and understand why `public` is conservative while `trusted` is dangerous.

This is intentionally **not** the “which NIC is bound where?” lab. That operational view lives in **Lab 60 — Inspect Active Firewall Zones**, where we emphasize interface bindings under load. Here, you build the **vocabulary** so those bindings make sense.

> **Lab safety note:** Read-only until Task 6’s optional scratch notes. Task 6 **Cleanup** removes `/tmp` artifacts only — no firewall mutations in this catalog lab.

---

## Concept: Zones Are Labels for “How Much Do I Trust This Network?”

```
   ┌────────────────────────────────────────────────────────────┐
   │  firewalld                                                  │
   │   ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
   │   │ public  │   │ internal│   │ trusted │   ...           │
   │   │ zone    │   │ zone    │   │ zone    │                 │
   │   └────┬────┘   └────┬────┘   └────┬────┘                 │
   │        │ services/   │             │                       │
   │        │ ports       │             │                     │
   └────────┼─────────────┼─────────────┼──────────────────────┘
            │             │             │
            ▼             ▼             ▼
      (runtime nft     same kernel    same kernel
       rulesets)       hooks          hooks

An interface (or source) mapped to a zone inherits that zone's rules.
Default zone applies when nothing more specific matches.
```

> **Why this matters:** Misplaced trust costs breaches. Mapping the **wrong** interface into `trusted` is equivalent to leaving the drawbridge down. Cataloging zones trains your gut for later interface moves.

---

## 📜 Why firewalld Zones Exist — The Story

Before dynamic firewalls, administrators edited static scripts and reloaded entire tables when anything changed — high risk, slow change windows, and painful rollbacks. The `firewalld` daemon introduced a **runtime** configuration API so rules could be inserted and removed without rewriting everything by hand.

Zones solve a second problem: **humans do not think in port tuples** during a conference call — they think “this link is the management network” or “that Wi-Fi is hostile.” A zone is a reusable label (`internal`, `dmz`, `block`) that bundles services, ports, masquerading, and ICMP filters into one named policy.

Red Hat Enterprise Linux has shipped `firewalld` as the default firewall service since **RHEL 7**, layering atop the kernel’s evolving backends (`iptables` → `nftables`). The exam does not ask you to memorize release trivia; it asks you to **list**, **compare**, and **apply** the right zone to the right interface when a scenario demands it.

> **The point of the story:** Zones are not magic — they are curated defaults so you can say “treat this NIC like a DMZ” in one command instead of forty port rules.

---

## 👪 The firewalld Zone Family — Who Lives There

### By read-only listing commands

| Intent | Command |
|---|---|
| All zone names | `firewall-cmd --get-zones` |
| Default zone name | `firewall-cmd --get-default-zone` |
| Everything at once (verbose) | `firewall-cmd --list-all-zones` |
| One zone deep dive | `firewall-cmd --info-zone=public` |

### By trust posture (typical pattern)

| Zone | Posture | Typical use |
|---|---|---|
| `drop` | Deny-first | Honeypots, hostile uplinks |
| `block` | Inbound refused | Similar intent, different ICMP behavior |
| `public` | Conservative default | Untrusted coffee-shop / internet NIC |
| `internal` | More open | LAN servers you administer |
| `trusted` | Almost wide open | Dangerous — lab only |
| `dmz` | Demilitarized pattern | Internet-facing app tier |

### By “where does this interface live?” (preview only)

| Command | Why defer detail |
|---|---|
| `firewall-cmd --get-active-zones` | **Lab 60** spends a full hour there |
| `nmcli` | ties to NetworkManager — optional cross-check |

> **The point of the family tree:** This lab memorizes **names + defaults**. Lab 60 memorizes **bindings**.

---

## 🔬 The Anatomy of `firewall-cmd --list-all-zones` — In One Diagram

```
$ firewall-cmd --list-all-zones | sed -n '1,40p'
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  ...
```

Read it as:

```
ZONE (active?)
  ├─ target            ← default ACCEPT/DROP semantics inside zone
  ├─ interfaces: ...   ← NICs currently inheriting this zone (runtime)
  ├─ sources: ...      ← IP ranges mapped here (if any)
  ├─ services: ...     ← named opens (ssh, http, ...)
  └─ ports: ...        ← raw port/protocol openings
```

> **Reading rule:** `(active)` only prints on zones that currently have **at least one binding** (interface or source). Inactive zones still exist as templates.

---

## 📚 firewalld Zone Catalog Reference Table

| Task | Command | Notes |
|---|---|---|
| Show daemon | `systemctl status firewalld --no-pager` | Confirms D-Bus API availability |
| List names | `firewall-cmd --get-zones` | Alphabet soup → sorted mental map |
| Default | `firewall-cmd --get-default-zone` | Often `public` on servers |
| Full matrix | `firewall-cmd --list-all-zones` | Large output — pipe to `less` |
| Single zone | `firewall-cmd --info-zone=dmz` | Services/ports/target in one block |
| Permanent peek | `firewall-cmd --permanent --get-default-zone` | Compare to runtime in later labs |

> **Rule one of exploration:** Always know whether you are looking at **runtime** (now) or **permanent** (after reload/boot). This lab stays runtime-first.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | You must instantly know how to **enumerate** zones before you **change** them. |
| **RHCE candidate** | Ansible `zone:` parameters assume you already understand trust bundles. |
| **SRE / Platform** | Incidents begin with “what zone is this NIC in?” — vocabulary first. |
| **DevOps** | Base AMIs often hardcode `public`; services fail mysteriously until you read `services:` line. |
| **AI / MLOps** | Separate admin SSH from GPU fabric NICs mentally before you open ports. |

---

## 🔧 The 6 Tasks

> Build the **mental card deck** of zones — no persistent mutations.

---

### Task 1 — Confirm firewalld is running and responsive

**Purpose:** Verify the D-Bus interface before deeper listings.

```bash
sudo -i
systemctl is-active firewalld
firewall-cmd --state
firewall-cmd --version
```

**Human-Readable Breakdown:** `active` + `running` means listing commands will succeed. `--version` documents the lab host for screenshots.

**Reading it left to right:** `firewall-cmd --state` is the quickest boolean for automation scripts too.

**The story:** If the daemon is stopped, every later command errors — fix upstream first.

**Expected output:**

```text
active
running
0.9.9
```

**Switches**

| Token | Meaning |
|---|---|
| `systemctl is-active` | Prints single-word state |
| `--state` | `firewalld`-specific running check |
| `--version` | User/daemon version banner |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Failed to connect to dbus` | Start service: `systemctl start firewalld` |
| `Authorization failed` | Use `sudo` / `sudo -i` |

---

### Task 2 — Enumerate every zone name and highlight the default

**Purpose:** Produce the sorted set of zone identifiers your brain will pattern-match on exams.

```bash
firewall-cmd --get-zones | tr ' ' '\n' | sort | column
echo "DEFAULT=$(firewall-cmd --get-default-zone)"
```

**Human-Readable Breakdown:** `get-zones` prints a single space-separated line — converting to newline aids reading.

**Reading it left to right:** `tr` swaps delimiters; `sort` alphabetizes; `column` pretty-prints single-column text harmlessly.

**The story:** Alphabetical order has no security meaning — it simply helps your eye catch `dmz` vs `drop`.

**Expected output:**

```text
block
dmz
drop
...
public
...
DEFAULT=public
```

**Switches**

| Token | Meaning |
|---|---|
| `--get-zones` | All configured zone names |
| `--get-default-zone` | Fallback zone for unmapped traffic |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty output | Extremely broken config — reinstall `firewalld` files from RPM |

---

### Task 3 — Core A: `list-all-zones` and compare `public` vs `trusted`

**Purpose:** See the **service bundles** side-by-side — this is the heart of the lab.

```bash
firewall-cmd --list-all-zones | less
firewall-cmd --info-zone=public
echo "====="
firewall-cmd --info-zone=trusted
```

**Human-Readable Breakdown:** First command is the atlas; the `info-zone` pair is the zoomed inset for extremes.

**Reading it left to right:** Notice `services:` lines differ wildly — that is trust expressed as names.

**The story:** `trusted` is not “friendlier public” — it is “assume goodwill.” Treat it like root password printed on a sticky note.

**Expected output:**

```text
public (active)
  services: cockpit dhcpv6-client ssh
...
trusted
  services:
...
```

**Switches**

| Token | Meaning |
|---|---|
| `--list-all-zones` | Every zone’s runtime configuration |
| `--info-zone=NAME` | Focused stanza for one zone |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Pager confusion in `less` | Press `q` to quit |
| `public` not marked active | Normal if no interface mapped — still valid template |

---

### Task 4 — Core B: Compare `internal`, `dmz`, and `work` intent

**Purpose:** Train scenario recognition — “LAN server” vs “internet app tier” vs “corporate desk.”

```bash
for z in internal dmz work; do
  echo "######## $z ########"
  firewall-cmd --info-zone="$z"
done
```

**Human-Readable Breakdown:** Loop prints three comparable stanzas without hand-copying commands.

**Reading it left to right:** Bash `for` iterates strings; each iteration is a focused `info-zone`.

**The story:** Exam stories rarely say “use dmz” outright — they describe traffic patterns that **map** to these bundles.

**Expected output:**

```text
######## internal ########
internal
  target: default
  services: cockpit dhcpv6-client mdns samba-client ssh
...
######## dmz ########
...
```

**Switches**

| Token | Meaning |
|---|---|
| `for z in ...` | Repeat body for each zone label |
| `--info-zone=$z` | Variable expansion to each name |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `no such zone` typo | Re-run `--get-zones` to copy exact spelling |

---

### Task 5 — Edge case: services vs ports vs masquerade flags

**Purpose:** Read advanced keys you will later modify — masquerade, forward, icmp blocks.

```bash
firewall-cmd --info-zone=external | egrep 'services|ports|masquerade|forward'
firewall-cmd --query-masquerade --zone=external && echo "masq on (runtime)" || echo "masq off"
```

**Human-Readable Breakdown:** `external` often demonstrates NAT-ish toggles — perfect teaching zone.

**Reading it left to right:** `egrep` filters to interesting lines; `--query-masquerade` returns boolean exit status printed humanly.

**The story:** If you cannot read masquerade here, `firewall-cmd --add-masquerade` will feel like voodoo later.

**Expected output:**

```text
  services: ssh
  ports:
  masquerade: yes
  forward-ports:
  forward: yes
masq on (runtime)
```

**Switches**

| Token | Meaning |
|---|---|
| `--info-zone=external` | Focus external zone |
| `--query-masquerade --zone=` | True/false check for SNAT-ish behavior |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `masq off` on your VM | Normal — zones differ by policy; pick another with `yes` or note state |

---

### Task 6 — Capstone: build a zone comparison cheat-sheet file, then delete it

**Purpose:** Summarize default zone, active markers, and three chosen zones into a ticket-ready snippet.

```bash
OUT=/tmp/zones-lab-summary.txt
{
  date
  echo "DEFAULT=$(firewall-cmd --get-default-zone)"
  firewall-cmd --get-active-zones || true
  for z in public internal dmz; do
    echo "######## $z ########"
    firewall-cmd --info-zone="$z" || true
  done
} | tee "$OUT"

wc -l "$OUT"
```

**Human-Readable Breakdown:** This is your personal **flashcard file** for spaced repetition — then you delete it to leave the host clean.

**Reading it left to right:** `get-active-zones` is a teaser for Lab 60; here it only documents state.

**The story:** Engineers who can produce this file in 60 seconds get fewer follow-up questions from security reviewers.

**Expected output:**

```text
DEFAULT=public
public
  interfaces: eth0
...
######## public ########
...
```

**Switches**

| Token | Meaning |
|---|---|
| `tee "$OUT"` | Write summary and mirror to stdout |
| `get-active-zones` | Maps interfaces → zone (preview) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Permission denied on `/tmp` | Use root shell (`sudo -i`) |

**Cleanup**

```bash
rm -f /tmp/zones-lab-summary.txt
```

---

## 🔍 firewalld Zone Exploration Decision Guide

```
Need to learn zones?
  │
  ├── "What exists at all?"
  │       └── firewall-cmd --get-zones
  │
  ├── "Which is default if nothing matches?"
  │       └── firewall-cmd --get-default-zone
  │
  ├── "Show me everything (big)"
  │       └── firewall-cmd --list-all-zones | less
  │
  ├── "Zoom one zone"
  │       └── firewall-cmd --info-zone=NAME
  │
  ├── "Which NIC is where?"   ← deeper ops track
  │       └── Lab 60: active-firewall-zones
  │
  └── "Change bindings/services/ports"
          └── Labs 57–59, 61
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Verify `firewalld` active and `firewall-cmd --state` running
- [ ] 02 Print sorted zone list + echo default zone
- [ ] 03 Read full atlas in `less`, compare `public` vs `trusted` with `info-zone`
- [ ] 04 Loop `info-zone` for `internal`, `dmz`, `work`
- [ ] 05 Inspect `external` masquerade/forward hints with filtered grep
- [ ] 06 Generate `/tmp` summary with `tee`, verify line count, delete in Cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Confusing this lab with interface audit | Skipping vocabulary | Do Lab 60 next for bindings |
| Trusting `(active)` without reading why | Misunderstood NIC mapping | Read `interfaces:` under that zone |
| Assuming `trusted` is “easy mode” | Over-open host | Treat `trusted` like explosives |
| Piping huge output blindly | Lost context | Use `less` / `info-zone` |
| Editing zones here | Out of scope churn | Stay read-only except `/tmp` note |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Be able to recite how to list zones and explain `public` vs `internal` trust difference in one sentence each.

**RHCE candidate**
- Relate `firewalld` zone to Ansible `zone` parameter and `immediate` vs `permanent` flags (future labs).

**SRE / Platform interview**
- Describe how you would **prove** a host’s zone catalog matches golden image expectations.

**DevOps**
- Bake `firewall-cmd --list-all-zones` output into AMI tests as a drift detector.

**AI / MLOps**
- Map data ingestion NICs into stricter zones than management — vocabulary enables the architecture conversation.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [active-firewall-zones](https://github.com/kelvintechnical/active-firewall-zones) | **Operational** interface→zone audit (pairs with this vocabulary lab) |
| [default-firewall-zone](https://github.com/kelvintechnical/default-firewall-zone) | Changing default zone safely |
| [reassign-interfaces-zones](https://github.com/kelvintechnical/reassign-interfaces-zones) | Moving NICs between zones |
| [inspecting-iptables](https://github.com/kelvintechnical/inspecting-iptables) | Low-level chain view under the daemon |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
