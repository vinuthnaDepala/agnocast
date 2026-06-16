# Contribution Report

---

## 🔁 Reproduction Process

### Environment Setup
- Cloned the `agnocast` repository from GitHub
- Built kernel module using the project’s build system (`make` in `agnocast_kmod`)
- Loaded the module into the kernel using `insmod`
- Set up a test environment with:
  -> One topic
  -> 1 topic and also with multiple publishers writing entries to the RB-tree backed topic storage
  -> One subscriber consuming messages via `AGNOCAST_RECEIVE_MSG_CMD`

#### Issues faced
- Initial build failed due to missing kernel headers
  -> Fixed by installing matching Linux kernel headers for the running kernel version
- Debugging kernel logs required using `dmesg -w` to observe runtime behavior

---

### Steps to Reproduce
1. Create a topic using the agnocast user-space interface
2. Spawn multiple publishers and publish a burst of messages (> MAX_RECEIVE_NUM entries)
3. Create a subscriber and call `AGNOCAST_RECEIVE_MSG_CMD` repeatedly
4. Observe the returned entries across multiple ioctl calls
5. Notice:
   -> Repeated scanning of the same entries in the RB-tree
   -> Inconsistent or repeated message delivery
   -> Progress stalling when entries are filtered or invalid publishers are encountered

---

### Branch Link
https://github.com/vinuthnaDepala/agnocast/tree/optimize-receive-msg-ioctl

---

## 🧠 Solution Approach

### Implementation Plan (UMPIRE)

#### Understand
The issue occurs in `receive_msg_core()` inside `agnocast_ioctl.c`, where the subscriber’s cursor (`latest_received_entry_id`) is only updated based on **returned entries**, not **all visited entries during RB-tree traversal**.

As a result:
- Entries that are filtered ( due to `ignore_local_publications` or exited processes) are not accounted for in cursor progression
- If missing publisher info, the function returns early and prevents cursor advancement
- Causes repeated scanning of the same RB-tree region and inconsistent pagination behavior

---

#### Match
This follows a standard kernel pagination pattern using:
- RB-tree ordered traversal (`rb_first`, `rb_next`, `rb_last`)
- Cursor-based iteration (`entry_id` tracking)

Correct implementations in similar systems:
- Advance iteration state based on **visited nodes**, not just emitted output
- Ensure filtering does not affect traversal progress
- Avoid early returns that prevent cursor updates unless truly fatal

---

#### Plan
1. Introduce a `last_seen_entry_id` variable to track traversal progress independent of output batching
2. Update `last_seen_entry_id` on every RB-tree iteration step
3. Replace cursor update logic:
   - From: last returned entry ID
   - To: last visited entry ID
4. Replace fatal early returns in non-critical failure cases (e.g., missing publisher) with `continue`
5. Ensure cursor (`latest_received_entry_id`) is always updated even when:
   - No entries are returned
   - Entries are filtered out

---

#### Implement
- Will implement in Phase III
- Changes will be applied to:
  - `agnocast_kmod/agnocast_ioctl.c`
  - Specifically `receive_msg_core()` function

---

#### Review
- Ensure no early returns inside RB-tree traversal loop unless strictly fatal
- Confirm cursor always advances monotonically
- Validate locking correctness under `wrapper->topic_rwsem`
- Ensure changes do not break RB-tree ordering assumptions
- Follow kernel coding style (clear control flow, minimal complexity in hot path)

---

#### Evaluate
1. **Basic functionality test**
   - Publish more than `MAX_RECEIVE_NUM` entries
   - Verify no duplicates or missing entries across multiple ioctl calls

2. **Filtering test**
   - Enable `ignore_local_publications`
   - Ensure system still makes forward progress without infinite re-scanning

3. **Fault tolerance test**
   - Simulate missing or exited publishers
   - Ensure ioctl does not get stuck or repeatedly fail on same entries

4. **Regression check**
   - Verify no impact on unrelated ioctl commands or topic operations