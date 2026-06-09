# Group Chats

Status: **deferred** — detailed design postponed until direct messaging works end to end. Directional decisions are recorded below so future design starts from them, and nothing in the v1 protocol may contradict them.

## Decided direction

- **Client-side fan-out.** A group is a purely client-side construct: the roster exists only on members' devices, E2E. Sending to a group = N−1 ordinary pairwise deliveries through standard sealed envelopes. Servers never learn that groups exist — group traffic is indistinguishable from direct traffic; no new server code, no new metadata.
  - The group-mailbox alternative (group as a pseudo-user, members as its "devices") was considered and rejected: the hosting server sees group size and activity, and member removal is broken under a shared identity key (an excluded member can read forever; only full group re-creation helps).
- **Group token pools.** Each member issues one pool of one-time delivery tokens to the group (registered as usual, distributed via E2E group-control messages) instead of O(N²) pairwise stashes. Concurrent senders may collide on a token — same optimistic retry as multi-device (`DeviceSync.md`). Member removal: everyone revokes the pool tokens they issued to the group and broadcasts fresh pools.
- **Multi-admin governance.** The roster and the admin set are signed by any current admin's identity key; the creator is the first admin and can appoint others. The group survives the creator's account loss (identity keys are unrecoverable, see `DeviceSync.md`).
- **Target size ≤ ~100 members.** Fan-out over I2P is realistic at this scale (parallel sends, retries). Large broadcast channels are a separate future concept and must not be solved by stretching groups.
- **No batch-send API.** A batch endpoint would turn the statistical timing correlation at the sender's server into an explicit, reliable list of linked destination servers. Clients send individually with their own pacing/jitter policy.

## To design later (when groups are picked up)

- Join bootstrap: the inviter relays E2E token bundles between the new member and existing members (one hop, join time only).
- Roster conflict resolution between concurrent admin updates.
- Message ordering across members (best-effort client-side ordering).
- Attachment fan-out cost (×N uplink) and possible mitigations.
- Group calls.
