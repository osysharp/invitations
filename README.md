# Osysharp.Invitations

Bring a person into an app by their **address**, with what they will be — a role, and optionally the organisation they
hold it in — fixed when the invitation is sent. Mailed, reminded, lapsing, revocable and resendable, as a durable run.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.UserAccounts@1;
use Osysharp.Organizations@0;
use Osysharp.Invitations@2;
```

```osy
using Osysharp.Identity;
using Osysharp.Invitations;

[Role] enum AppRole { Authenticator, Agent, Admin, CompanyAdmin }
policy IsAdmin => RoleGrant.Any(g => g.Member == user && g.Role == AppRole.Admin);

policy MayInvite => IsAdmin;                 // who invites (and revokes, and resends)
policy ManagesOrganizations => IsAdmin;      // Osysharp.Organizations' one question

// How the app's roles read to people — what the invite form offers. A role not listed is never offered, and one held
// only inside an organisation is offered only on an invitation into one.
app.Roles = new RolesSetup { Offered = [
  new RoleChoice { Role = AppRole.Agent, Label = "Agent", Description = "Works tickets." },
  new RoleChoice { Role = AppRole.Admin, Label = "Admin", Description = "Runs the desk." },
  new RoleChoice { Role = AppRole.CompanyAdmin, Label = "Company admin", Description = "Looks after their company's people.",
                   HeldIn = RoleScope.Organization },
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
- The page **shows** the invited address, read from the link — it is never typed. When the inviter ticked "They may
  join with another address" (`Invite(…, anotherAddress: true)`, off by default), the newcomer may choose another one:
  a confirmation link is mailed to it, and only that link makes the account.
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

To add one paragraph under the button without rewriting the mail, override only `Postscript`:

```osy
public class MyInvitationMails : InvitationMails {
  public override string Postscript(Invitation invitation) { return "Questions? Our community is at https://example.com/community"; }
}
```

An organisation's own admin may be allowed to invite into it, as an ordinary grant:

```osy
partial entity Invitation {
  security { allow read, create, update when IsAuthenticated where Organization != null && HoldsRoleIn(Organization, AppRole.CompanyAdmin); }
}
```

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.Organizations;
using Osysharp.Invitations;
using Osysharp.Mail;
using Osysharp.Workflow;

[Test(Team)]
void a_newcomer_makes_an_account_for_the_invited_address_through_the_link_and_holds_the_role() {
  var token = Invited("nina@desk.test", AppRole.Agent);
  Assert.Equal("", JoinByInvitation(token, "Nina@Desk.test", "Nina Alvarez", "a-long-password-1"));
  string ticket = "";
  runas (AuthBootstrap) { ticket = SignIn("nina@desk.test", "a-long-password-1").Ticket; }
  Assert.NotEqual("", ticket);
  runas (Nina) {
    var me = User.Single(u => u.Email == "nina@desk.test");
    Assert.Equal("Nina Alvarez", me.DisplayName);
    Assert.NotNull(me.EmailVerifiedAt);
    Assert.True(RoleGrant.Any(g => g.Member == me && g.Role == AppRole.Agent));
  }
  runas (Sam) { Assert.Equal(InvitationStatus.Accepted, Invitation.Single(i => i.Email == "nina@desk.test").Status); }
}
```

From [`tests/accept.test.osy`](tests/accept.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.Invitations;
using Osysharp.Mail;
using Osysharp.Workflow;

[Test(AddressDesk)]
void the_links_key_shows_the_invited_address_to_a_newcomer() {
  var tail = InvitedWith("nina@desk.test", false);
  var shown = InvitationByKey(InvitationKey.From(KeyOf(tail)));
  Assert.Equal("nina@desk.test", shown.Email);
  Assert.True(shown.Open);
  Assert.False(shown.AnotherAddressAllowed);
}
```

From [`tests/another-address.test.osy`](tests/another-address.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;
using Osysharp.Organizations;
using Osysharp.Invitations;
using Osysharp.Mail;
using Osysharp.Workflow;

[Test(Desk)]
void an_invitation_is_mailed_once_with_its_link_after_it_is_saved() {
  Invitation? sent = null;
  runas (Sam) { sent = Invite("  Nina@Desk.TEST ", AppRole.Agent, note: "Welcome aboard"); }
  Workflow.Settle(sent!);
  runas (AuthBootstrap) {
    Assert.Equal(1, InvitationMailsTo("nina@desk.test"));
    Assert.NotEqual("", InvitationTokenFor("nina@desk.test"));
  }
  runas (Sam) {
    var inv = Invitation.Single(i => i.Email == "nina@desk.test");
    Assert.Equal(InvitationStatus.Pending, inv.Status);
    Assert.Equal(1, inv.Sends);
    Assert.NotNull(inv.ExpiresAt);
  }
}
```

From [`tests/invite.test.osy`](tests/invite.test.osy).

## Learn more

- [The Osysharp.Invitations reference](https://osysharp.com/reference/security/invitations-kit/)
- [Osysharp.Invitations in the package catalogue](https://osysharp.com/packages/#osysharp-invitations)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
