# USERS

Two roles, two dedicated apps.

## admin

Operates the shared corpus. Signs into the admin app. Uploads documents visible to every team member. Manages users (sees who has signed in, can revoke). Reads the audit log of agent activity if needed.

## user

Operates own corpus + reads shared. Signs into the user app. Uploads own documents (visible only to self), asks the agent questions over shared + own scope, downloads their chat history.

## Role mechanism

Role is determined by which app the user signed into, not by an email allowlist. The admin app is reached via tighter network access (admin-only URL / VPN segment / internal subnet); the user app is reached via the team-wide internal network. Anyone who can reach the admin app and successfully complete Google OAuth becomes admin for that session. Anyone who reaches the user app becomes user.

Network is the gate. Self-hosting on a private network means the access perimeter is enforced at the network layer; OAuth identifies the human, the app endpoint encodes the role.
