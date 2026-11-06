# Feature Specification: User Accounts

**Feature Branch**: `001-user-accounts`

**Created**: 2026-11-06

**Status**: Draft

**Input**: User description: "user accounts"

## Clarifications

### Session 2026-11-06

- Q: Can someone who is not signed in see members' public profiles? → A: No. Only signed-in
  members can see public profiles; visitors see nothing about members.
- Q: Should the 5-failure block apply to the account everywhere, or only to attempts from the same
  device or network? → A: Only to attempts on that account from the same device or network; the
  owner can still sign in from elsewhere.
- Q: When an administrator deactivates a member, should other members still see that person's
  public profile? → A: Yes, but it is clearly marked as deactivated.
- Q: Should the system keep a permanent record of administrator actions that administrators can
  view? → A: Yes. Record each deactivation and reactivation (which administrator, which account,
  when), viewable by administrators.
- Q: Must display names be unique? → A: No. Duplicates are allowed; the neighborhood area helps
  tell members apart.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register, Sign In, and Sign Out (Priority: P1)

A neighbor who wants to lend or borrow tools creates an account with their email address, a
password, and a display name. They can then sign in, use LendLoop as themselves, and sign out
when they are done, especially on a shared device.

**Why this priority**: Every other LendLoop feature (listing tools, requesting loans) needs to know
who the person is. Without accounts, nothing else can be built or secured.

**Independent Test**: Register a new account, sign out, sign back in with the same credentials, and
confirm that signed-in-only pages are reachable while signed in and blocked after signing out.

**Acceptance Scenarios**:

1. **Given** no account exists for an email address, **When** a visitor registers with that email,
   a valid password, and a display name, **Then** an active account is created and the visitor is
   signed in.
2. **Given** an account already exists for an email address (in any letter case), **When** a
   visitor tries to register with that email, **Then** registration is refused and no second account
   is created.
3. **Given** an active account, **When** its owner signs in with the correct email and password,
   **Then** they are signed in and see their own display name.
4. **Given** an active account, **When** someone signs in with a wrong password or an unknown email,
   **Then** sign-in fails with the same generic message in both cases.
5. **Given** a signed-in member, **When** they sign out, **Then** their session ends and reusing it
   no longer grants access.
6. **Given** a visitor who is not signed in, **When** they try to open a members-only page or
   action, **Then** they are refused and asked to sign in.

---

### User Story 2 - Manage My Profile and Password (Priority: P2)

A signed-in member views and edits their own profile (display name and neighborhood area) and
changes their password when they want to.

**Why this priority**: Members need to keep their identity accurate and their account secure, but
the app is usable for a first demo with only registration and sign-in.

**Independent Test**: Sign in, change the display name and neighborhood area, change the password,
then sign in again with the new password and confirm the profile changes persisted.

**Acceptance Scenarios**:

1. **Given** a signed-in member, **When** they update their display name or neighborhood area,
   **Then** the change is saved and shown wherever their public profile appears.
2. **Given** a signed-in member, **When** they change their password and supply their current
   password correctly, **Then** the new password works for future sign-ins and the old one does not.
3. **Given** a signed-in member, **When** they try to change their password with a wrong current
   password, **Then** the change is refused.
4. **Given** member A is signed in, **When** they try to view or change member B's private profile
   details or password, **Then** the action is refused and nothing about member B changes.
5. **Given** a member changes their password, **When** another session of theirs (for example, on
   another device) is used afterwards, **Then** that other session no longer grants access.
6. **Given** a visitor who is not signed in, **When** they try to view any member's public profile,
   **Then** the request is refused.

---

### User Story 3 - Administrators Manage Members (Priority: P3)

An administrator sees the list of members and can deactivate an account (for example, after abuse
reports) or reactivate it later.

**Why this priority**: Needed to keep the community safe once real neighbors use it, but not
required to start lending between a handful of trusted users.

**Independent Test**: As an administrator, deactivate a member and confirm they can no longer sign
in or use an existing session; reactivate them and confirm they can sign in again. As a regular
member, confirm every administrator action is refused.

**Acceptance Scenarios**:

1. **Given** a signed-in administrator, **When** they open the member list, **Then** they see each
   member's display name, email address, role, status, and registration date.
2. **Given** an active member, **When** an administrator deactivates them, **Then** the member can
   no longer sign in, and any session they already had stops granting access.
3. **Given** a deactivated member, **When** an administrator reactivates them, **Then** the member
   can sign in again with their existing password.
4. **Given** a signed-in regular member, **When** they try to view the member list or deactivate or
   reactivate any account, **Then** the action is refused.
5. **Given** a signed-in administrator, **When** they try to deactivate their own account, **Then**
   the action is refused.
6. **Given** a deactivated member, **When** another signed-in member views their public profile,
   **Then** it shows their display name and neighborhood area, clearly marked as deactivated.
7. **Given** an administrator has deactivated and later reactivated a member, **When** any
   administrator views the record of administrator actions, **Then** both actions appear with who
   performed them, on which account, and when.
8. **Given** a signed-in regular member, **When** they try to view the record of administrator
   actions, **Then** the request is refused.

---

### Edge Cases

- A visitor registers with an email that differs from an existing one only by letter case or
  surrounding spaces: it is treated as the same email and refused.
- The same registration is submitted twice in quick succession (for example, a double click or a
  retried request): exactly one account is created.
- Someone repeatedly fails to sign in to the same account: after 5 failures within 15 minutes from
  the same device or network, further attempts on that account from that device or network are
  refused for 15 minutes, even with the correct password.
- Someone deliberately fails to sign in to a neighbor's account: the neighbor can still sign in
  from their own device or network.
- A deactivated member tries to sign in: they get the same generic failure as a wrong password.
- A display name is empty, only whitespace, or longer than 50 characters: it is refused.

## Requirements *(mandatory)*

### Functional Requirements

**Registration and sign-in**

- **FR-001**: Visitors MUST be able to register with an email address, a password, and a display
  name.
- **FR-002**: The system MUST treat email addresses as unique, ignoring letter case and surrounding
  whitespace, and MUST guarantee that retried or simultaneous registrations never create two
  accounts for the same email.
- **FR-003**: A new account MUST be active immediately on registration, and the new member MUST be
  signed in automatically.
- **FR-004**: The system MUST require passwords of at least 12 and at most 128 characters, and MUST
  NOT impose other composition rules (such as required symbols).
- **FR-005**: The system MUST store passwords only in a form from which the original password
  cannot be recovered.
- **FR-006**: Members MUST be able to sign in with email and password, and sign out.
- **FR-007**: Failed sign-ins MUST show one generic message that does not reveal whether the email
  exists, the password was wrong, or the account is deactivated.
- **FR-008**: After 5 failed sign-in attempts on one account from the same device or network within
  15 minutes, the system MUST refuse further attempts on that account from that device or network
  for 15 minutes. Attempts on the same account from other devices or networks MUST NOT be affected.
- **FR-009**: Sessions MUST expire after 7 days without activity, and signing out MUST end the
  session immediately on the server side.

**Profile and password**

- **FR-010**: Members MUST be able to view and edit their own display name (1–50 characters after
  trimming) and neighborhood area (optional, up to 100 characters). Display names do not need to
  be unique; two members MAY have the same display name.
- **FR-011**: Members MUST be able to change their password by supplying their current password;
  doing so MUST end all of their other sessions.
- **FR-012**: A member's public profile MUST show only their display name and neighborhood area,
  and MUST be visible only to signed-in members. Visitors who are not signed in MUST NOT be able to
  see any information about members. Email addresses, roles, and password data MUST NOT be
  visible to other members. The only account status other members see is the deactivated marker
  described in FR-017.
- **FR-013**: Members MUST NOT be able to view another member's private details or change anything
  on another member's account.

**Roles and administration**

- **FR-014**: Every account MUST have exactly one role: member or administrator. New registrations
  are always members.
- **FR-015**: Administrators MUST be able to list all accounts with display name, email, role,
  status (active or deactivated), and registration date.
- **FR-016**: Administrators MUST be able to deactivate and reactivate accounts, except their own.
- **FR-017**: Deactivating an account MUST immediately end its sessions, and a deactivated account
  MUST NOT be able to sign in. Its public profile MUST remain visible to signed-in members, clearly
  marked as deactivated; reactivating the account removes the marker.
- **FR-018**: Only administrators MUST be able to perform administrator actions; every such attempt
  by a member or visitor MUST be refused.
- **FR-019**: All of the access rules in FR-012, FR-013, and FR-018 MUST be enforced by the system
  itself, not only by hiding options in the user interface.
- **FR-020**: The system MUST permanently record every deactivation and reactivation with the
  administrator who performed it, the affected account, the action, and when it happened. The
  record MUST be saved together with the status change, so neither can exist without the other.
  Administrators MUST be able to view these records, newest first; nobody can edit or delete them.

### Key Entities

- **Account**: A person registered in LendLoop. Holds email (unique, private), protected password,
  display name, neighborhood area, role (member or administrator), status (active or deactivated),
  and registration date.
- **Session**: Proof that an account is currently signed in on a device. Belongs to one account,
  has a last-activity time and expiry, and can be ended by sign-out, password change, or
  deactivation.
- **Account Status Change**: A permanent record of one administrator action. Holds the
  administrator who acted, the affected account, the action (deactivated or reactivated), and the
  time. Cannot be edited or deleted.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new visitor can register and reach the signed-in home page in under 2 minutes.
- **SC-002**: A returning member can sign in in under 30 seconds.
- **SC-003**: 100% of attempts by one member to read another member's private details or change
  their account, and 100% of administrator actions by non-administrators, are refused, as shown by
  automated tests for each such rule.
- **SC-004**: Sign-in failure responses are identical for existing and non-existing emails, so an
  observer cannot tell which emails have accounts.
- **SC-005**: After a deactivation or password change, previously active sessions are refused on
  their very next use.

## Assumptions

- LendLoop serves a single neighborhood community; there is no separation into multiple
  neighborhoods or groups. "Neighborhood area" is free text the member writes, such as a street or
  block, and is not verified.
- Email addresses are not confirmed in this feature; anyone can register with any address.
  Email confirmation is deferred until the main lending flow is complete.
- Password reset is out of scope for this feature and deferred until the main lending flow is
  complete. Until then, a member who forgets their password has no in-app way to recover access.
- The first administrator is created by the project operator outside the app (for example, a setup
  step). Promoting or demoting administrators in the app is out of scope for this feature.
- Members cannot change their email address or delete their own account in this feature; both can
  be added later.
- Sign-in with third-party identity providers is out of scope (constitution principle V).
- Profile photos, phone numbers, and member-to-member messaging are out of scope.
- This feature sends no email.
- Accessibility follows standard web expectations; no specific certification is targeted.
