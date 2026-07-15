# RQ-002: Create Admin User and Disable Root SSH

| Field           | Value                                                        |
|-----------------|--------------------------------------------------------------|
| Change ID       | CRQ-002                                                      |
| Title           | Create dedicated admin user; disable direct root SSH login   |
| Owner           | Hector Rodriguez                                             |
| Date submitted  | 2026-07-15                                                  |
| Scheduled start | 2026-07-15                                                  |
| Scheduled end   | 2026-07-15                                                  |
| Risk level      | High                                                         |
| Status          | In Progress                                                 |

---

## 1. Reason for Change

The `rhel-oe-build-lab` fleet (servera, serverb, serverc, serverd)
currently connects to managed hosts as `root` over SSH. This is a
deliberate lab-bootstrap convenience, but it is not an acceptable
long-term posture:

- Direct root SSH login bypasses the separation of privilege that
  `sudo` provides, so there is no per-operator audit trail.
- SSH brute-force attacks targeting `root` are constant background
  noise on any exposed host.
- The STIG and CIS benchmarks for RHEL both require `PermitRootLogin no`.

This change moves the lab to its intended access model:

1. A dedicated `admin` account is created on all hosts via the `users`
   role, carrying the operator SSH public key.
2. `baseline_sshd_permit_root_login` is overridden to `"no"` in
   `group_vars/all.yml` (its default is `"yes"` in
   `roles/baseline_hardening/defaults/main.yml`).
3. The inventory `ansible_user` is changed from `root` to `admin`.

---

## 2. Systems Affected

| Host    | Address      | Group      | Impact                              |
|---------|--------------|------------|-------------------------------------|
| servera | 10.0.0.101   | production | New user created; root SSH disabled |
| serverb | 10.0.0.74    | production | New user created; root SSH disabled |
| serverc | 10.0.0.56    | production | New user created; root SSH disabled |
| serverd | 10.0.0.36    | production | New user created; root SSH disabled |

Out of scope: the control node.

---

## 3. Risk Assessment

| Risk                                    | Likelihood | Impact | Mitigation                                                        |
|-----------------------------------------|------------|--------|-------------------------------------------------------------------|
| SSH key not deployed before root locked | Medium     | High   | Verify admin key login on all hosts before disabling root         |
| sudoers misconfiguration locks admin    | Low        | High   | Validate with `visudo -c` before and after; test `sudo` as admin  |
| One host missed during key deployment   | Low        | High   | Verify each host individually at the gate; do not proceed on fail |
| Inventory change breaks other playbooks | Low        | Medium | All plays target the `production` group, so the change is uniform |

**Risk rating: High.** Any mistake in this sequence can lock the
operator out of a host. The mitigation is strict sequencing with a
hard verification gate before root login is disabled.

---

## 4. Pre-Checks

Run on the control node and capture the output:

```bash
# All hosts reachable as root
ansible production -m ping

# Root SSH is currently permitted
ansible production -m shell -a 'sshd -T | grep -i permitrootlogin'
# expect: permitrootlogin yes

# Sudoers tree is valid on every host
ansible production -m shell -a 'visudo -c'

# The operator public key that will be deployed
cat id_ed25519_oelab.pub
```

---

## 5. Implementation Steps

Execute in this exact order. Do not disable root SSH until Step 3 is
confirmed clean on all four hosts. Keep the VirtualBox console
available for every VM throughout the change.

### Step 1: Add the admin user to `group_vars/all.yml`

Paste the actual contents of `id_ed25519_oelab.pub` as the key value.

```yaml
users_list:
  - name: admin
    comment: "Lab Admin Account"
    groups:
      - admin
      - wheel
    shell: /bin/bash
    ssh_authorized_keys:
      - "<contents of id_ed25519_oelab.pub>"
```

### Step 2: Create the account with the users role

Still connected as root:

```bash
ansible-playbook playbooks/01-baseline.yml --tags users --check --diff
ansible-playbook playbooks/01-baseline.yml --tags users
```

### Step 3: Verification gate (manual, do not skip)

```bash
for ip in 10.0.0.101 10.0.0.74 10.0.0.56 10.0.0.36; do
  echo "== $ip =="
  ssh -i ./id_ed25519_oelab -o StrictHostKeyChecking=accept-new admin@$ip 'id; sudo -l'
done
```

Every host must report `admin` in the group list (including `wheel`
and `admin`) and `(ALL) NOPASSWD: ALL` from `sudo -l`. If any host
fails or hangs, STOP. Root SSH is still enabled, so nothing is lost.
Diagnose before continuing.

### Step 4: Disable root SSH

Only after all four hosts passed Step 3. Add to `group_vars/all.yml`:

```yaml
baseline_sshd_permit_root_login: "no"
```

> Note: this variable currently lives in the `baseline_hardening`
> role (`baseline_sshd_*`) and is consumed by its sshd drop-in task.
> A planned future change (CRQ-004) will split sshd posture into a
> dedicated `ssh` role and rename it to `ssh_permit_root_login`.

Preview on one host, then apply fleet-wide (still connecting as root;
the existing connection survives the sshd restart):

```bash
ansible-playbook playbooks/01-baseline.yml --tags ssh --check --diff --limit servera
ansible-playbook playbooks/01-baseline.yml --tags ssh
```

### Step 5: Switch the inventory to the admin user

```ini
[all:vars]
ansible_user=admin
ansible_become=true
```

Leave the `ansible_host=` lines under `[production]` unchanged.

### Step 6: Confirm the control node still reaches all hosts

```bash
ansible production -m ping
ansible production -m shell -a 'whoami; sudo whoami'
# expect: admin, then root
```

### Step 7: Confirm root SSH is now blocked

```bash
ansible production -m shell -a 'sshd -T | grep -i permitrootlogin'
# expect: permitrootlogin no on all hosts

ssh -i ./id_ed25519_oelab root@10.0.0.101 'true'
# expect: Permission denied (publickey). This failure is the success condition.
```

---

## 6. Validation Steps

The dedicated validation playbook (`playbooks/03-validation.yml`) is
added in a later step of this build and does not exist yet. For this
change, validation is the manual evidence captured in Steps 3, 6, and
7:

- All four hosts pass the admin `id; sudo -l` gate.
- `ansible production -m ping` succeeds as `ansible_user=admin`.
- `sudo whoami` returns `root` on every host.
- `sshd -T` reports `permitrootlogin no` fleet-wide.
- A direct `ssh root@<host>` attempt is rejected.

Save the terminal output as evidence and reference it in Results.

---

## 7. Rollback Plan

If any host loses SSH access:

1. Open that VM's console in VirtualBox and log in locally as root.
2. Temporarily set `PermitRootLogin yes` in
   `/etc/ssh/sshd_config.d/00-baseline.conf`.
3. Restart sshd: `systemctl restart sshd`.
4. Reconnect from the control node and diagnose the root cause.
5. Once access is restored, fix the cause before re-applying.

Rollback is manual because the automation path itself depends on
working SSH. Keep the VM consoles reachable for the whole change window.

---

## 8. Communication Plan

- The inventory `ansible_user` changes from `root` to `admin`. Any
  local SSH config or scripts pointing at these hosts as `root` must
  be updated afterward.
- The operator public key must be in `group_vars/all.yml` before the
  window opens.

---

## 9. Results

| Field            | Value                                                                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Actual start     | 2026-07-15                                                                                                                                                 |
| Actual end       | 2026-07-15                                                                                                                                                 |
| Outcome          | Success - admin account created fleet-wide with key-based login and passwordless sudo; direct root SSH disabled; inventory switched to ansible_user=admin. |
| Hosts succeeded  | servera (10.0.0.101), serverb (10.0.0.74), serverc (10.0.0.56), serverd (10.0.0.36)                                                                        |
| Hosts deferred   | None                                                                                                                                                       |
| Attached output  | Manual verification captured below (no 03-validation.yml yet; full validation playbook is a later CRQ).                                                    |
| Git commit / tag | <6946859: output of `git rev-parse --short HEAD` after you commit this change>                                                                             |

Verification evidence:

- Step 3 gate (admin key login + sudo), all four hosts:
  `id` reported `uid=1001(admin)` with supplementary groups `wheel` and
  `admin`; `sudo -l` / `sudo -n true` returned NOPASSWD success on every
  host. No host failed the gate.
- Post-change sshd posture, all four hosts:
  `sshd -T | grep -i permitrootlogin` returned `permitrootlogin no`.
- Root login rejection confirmed:
  `ssh -i ./id_ed25519_oelab root@10.0.0.101 'true'` returned
  "Permission denied (publickey)", i.e. direct root SSH is refused.
- Control-node

---

## 10. References

- `roles/users/` — creates the admin account and sudoers drop-in
- `roles/baseline_hardening/` — owns the sshd drop-in and the
  `baseline_sshd_permit_root_login` default (pre-CRQ-004)
- `playbooks/01-baseline.yml` — applied with `--tags users` then
  `--tags ssh` during this change
