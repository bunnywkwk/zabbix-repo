# Master Task Baseline: Zabbix Observability Implementation

## Objective
Fulfill the "Observability requirements → Zabbix" PDF instructions. Turn requirement clusters into working Zabbix templates, test them against RHEL machines, and automate their deployment via Ansible.

## Our Core Engineering Principles
1. **Encounter Before Solving:** We take this one step at a time. We do not jump ahead. We face a problem, understand it, and then solve it.
2. **Do Not Over-Engineer:** We use the simplest, most secure native tools available.
3. **Honest Justification:** If a requirement cannot be achieved natively or securely, we stop, document exactly *why*, and provide legitimate limitations rather than forcing a bad workaround.
4. **Visual Documentation:** We will not overload documentation with text. We will use a brief explanation followed by a screenshot.

---

## Progress Tracker (Goal: 4 Clusters)

### [COMPLETED] Cluster 1: Host Health & Availability
- [x] **Research & Justify:** Chose Native Agent Checks to avoid root execution and third-party scripts.
- [x] **Template Creation:** Build the YAML blueprint for CPU, Memory, Service state, etc.
- [x] **Testing [ ] **Testing & Screenshots:** Screenshots:** Apply template to `Rhel9-Not-Hardened` and capture proof.
- [x] **Automated Import:** Write Ansible playbook to push to Zabbix Server.

### [IN PROGRESS] Cluster 2: Storage & Filesystem
- [ ] Research & Justify
- [ ] Template Creation
- [ ] Testing & Screenshots
- [ ] Automated Import

### [PENDING] Cluster 3: (To be selected from Scope)
- [ ] Research & Justify...

### [PENDING] Cluster 4: (To be selected from Scope)
- [ ] Research & Justify...

---

## Testing Documentation Format (For the Mentor)
*When we reach the testing phase for each cluster, I will instruct you to take these specific screenshots. You will paste them here to keep the text short and the proof visual.*

### Cluster 1 Testing Evidence
**1. Trigger Verification (Memory/CPU)**
*   **Action:** We temporarily lowered the threshold to force an alert.
*   **Result:** Zabbix successfully detected the threshold violation using native keys.
*   **Screenshot:** `[PLACEHOLDER: Insert image of Zabbix Dashboard showing the red CRITICAL alarm]`

**2. Service Status Verification (SSH)**
*   **Action:** We stopped the SSH service on the RHEL node to test the `systemd` tracker.
*   **Result:** Zabbix immediately logged the state change and fired the trigger.
*   **Screenshot:** `[PLACEHOLDER: Insert image of the Latest Data graph showing SSH state dropping]`
