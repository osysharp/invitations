# Osysharp.Invitations

Bring a person into an app by their **address**, with what they will be — a role, and optionally the organisation they
hold it in — fixed when the invitation is sent. Mailed, reminded, lapsing, revocable and resendable, as a durable run.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Accounts@0;
use Osysharp.Organizations@0;
use Osysharp.Invitations@0;
```

```osy
using Osysharp.Identity;
using Osysharp.Invitations;

[Role] enum AppRole { Authenticator, Agent, Admin }
policy IsAdmin => RoleGrant.Any(g => g.Member == user && g.Role == AppRole.Admin);

policy MayInvite => IsAdmin;                 // who invites (and revokes, and resends)
policy ManagesOrganizations => IsAdmin;      // Osysharp.Organizations' one question

// How the app's roles read to people — what the invite form offers. A role not listed is never offered.
app.Roles = new RolesSetup { Offered = [
  new RoleChoice { Role = AppRole.Agent, Label = "Agent", Description = "Works tickets." },
  new RoleChoice { Role = AppRole.Admin, Label = "Admin", Description = "Runs the desk." },
] };
```

Then, on a page only people who may invite reach:

```osy
InviteForm(canSend: MayInvite);
InvitationList(canChange: MayInvite);
```

`/invite/{token}` — the page the mail links to — is the kit's.

From code: `Invite("ada@acme.com", AppRole.Agent)`, `Invite(email, AppRole.CompanyAdmin, organization: acme)`,
`RevokeInvitation(inv)`, `ResendInvitation(inv)`.

## Who answers it

**Only the address it was sent to**, through one of two doors:

- **Somebody new** opens the mailed link and makes their account. The link works once, is stored only as a hash, dies
  when the invitation is revoked, resent or lapses, and makes an account **for the invited address and no other** —
  proven, since the mail was opened. Signing in is then the ordinary sign-in: an app that requires two-step sign-in of
  the role sets it up before any ticket.
- **Somebody who has an account** signs in — every step, the second factor included — and accepts in the app. Only an
  account whose address is the invited one, **and proven**, may: an account a stranger made with somebody else's address
  first cannot accept that person's invitation. The link does **not** accept for an existing account — holding a mailbox
  is not holding the account.

## What holds whatever the app grants

- What an invitation grants is fixed at send: its address, role and organisation are never edited.
- It never carries the sign-in flow's `Authenticator` role, and its inviter is whoever sent it.
- Accepting grants exactly that — the app role, or the membership and the role held in the organisation — once.

## Settings

```osy
app.Invitations = new InvitationsSetup {
  ValidFor = TimeSpan.FromDays(7), RemindAfter = TimeSpan.FromDays(3), RemindEvery = TimeSpan.FromDays(2),
  Mails = new MyInvitationMails(),    // derive from InvitationMails to write your own
};
```

An organisation's own admin may be allowed to invite into it, as an ordinary grant:

```osy
partial entity Invitation {
  security { allow read, create, update when IsAuthenticated where Organization != null && HoldsRoleIn(Organization, AppRole.CompanyAdmin); }
}
```
