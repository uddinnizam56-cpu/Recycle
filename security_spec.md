# PunarChakra Firestore Security Specification

## 1. Data Invariants
1. **Default Deny**: All paths under `/databases/{database}/documents` default to `allow read, write: if false`.
2. **PII Isolation**: Email, phone number, and street/area address are stored only inside `/users/{userId}/private/info` and restricted strictly to the document owner (`request.auth.uid == userId`) or verified admin. `/users/{userId}` holds only non-PII public recycling totals.
3. **Anti-Privilege-Escalation**: Users creating `/users/{userId}` can only set `role == 'customer'`; admin records in `/admins/{adminUid}` require verified bootstrap admin token (`nizamaazuddin@gmail.com`) or existing admin approval.
4. **Master Gate & Existence Checks**: `/users/{userId}/private/{docId}` verifies existence of parent `/users/{userId}` using `exists()`.
5. **Immutable Ownership & Temporal Integrity**: `ownerUid`, `uid`, and `createdAt` cannot change after creation; `createdAt` and `updatedAt` must equal `request.time`.
6. **Action-Based Updates & Terminal Locking**: Updates on `/pickup_orders/{orderId}` use explicit action field allowlists (`affectedKeys().hasOnly(...)`) and forbid owner edits once `status` is `'completed'` or `'cancelled'`.
7. **Query Enforcer**: All `allow list` rules explicitly evaluate `resource.data.ownerUid == request.auth.uid || isBootstrapAdmin()` without executing `get()` or `exists()` per document.

## 2. The "Dirty Dozen" Payloads (Expected: PERMISSION_DENIED)
1. **Unauthenticated read on `/users/u1`**: Missing `request.auth`.
2. **Self-assigned admin role on `/users/u2`**: `{ uid: "u2", role: "admin", ... }` created by non-admin.
3. **Shadow field injection on `/users/u2`**: `{ ..., isSuperAdmin: true }` rejected by `keys().hasOnly(...)`.
4. **Cross-user PII read on `/users/u1/private/info`**: Signed in as `u2`, reading `u1`'s private doc.
5. **Oversized ID poisoning on `/pickup_orders/<150-char-id>`**: Rejected by `isValidId(orderId)` (`size() <= 128`).
6. **Client-forged timestamp on `/pickup_orders/o1`**: `createdAt: "2020-01-01T00:00:00Z"` rejected by `createdAt == request.time`.
7. **Owner spoofing on `/pickup_orders/o1`**: Signed in as `u2`, setting `ownerUid: "u1"`.
8. **Updating terminal order `/pickup_orders/o_done`**: Attempting to mutate a `'completed'` order as non-admin.
9. **Mass status/amount tampering by customer on `/pickup_orders/o1`**: Attempting to overwrite `finalAmount` and `status: 'completed'` together without admin privileges.
10. **Non-admin write to `/scrap_rates/copper`**: Regular user attempting to raise scrap spot rates.
11. **Oversized string DoS on `/activity_logs/l1`**: Injecting >500 chars into `details` or >200 chars into `summary`.
12. **Email spoof admin claim**: Account with `email: "nizamaazuddin@gmail.com"` and `email_verified: false`.
