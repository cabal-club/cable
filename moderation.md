# Cable Moderation

Version: 1.0-draft8

Author: Alexander Cobleigh

## Abstract

This document describes the subjective moderation system designed to work in conjunction with
the network protocol "cable", weaving content moderation into its peer-to-peer group
chatrooms.

## Table of Contents

- [Abstract](#abstract)
- [1 Introduction](#1-introduction)
  * [1.1 Background](#11-background)
  * [1.2 Overview](#12-overview)
  * [1.3 Differences between actions, roles, moderators, and administrators](#13-differences-between-actions-roles-moderators-and-administrators)
    + [1.3.1 Roles](#131-roles)
    + [1.3.2 Roles as vouching](#132-roles-as-vouching)
    + [1.3.3 Actions and revocation](#133-actions-and-revocation)
  * [1.4 Design philosophy](#14-design-philosophy)
- [2 Conformance requirements](#2-conformance-requirements)
  * [2.1 Terminology and other Conventions](#21-terminology-and-other-conventions)
- [3 Definitions](#3-definitions)
- [4 Data model](#4-data-model)
  * [4.1 Channel context](#41-channel-context)
    + [4.1.1 The cabal context](#411-the-cabal-context)
  * [4.2 Roles](#42-roles)
    + [4.2.1 Moderation authority](#421-moderation-authority)
      - [4.2.1.1 Subjective authority](#4211-subjective-authority)
        * [4.2.1.1.1 Role transitivity](#42111-role-transitivity)
        * [4.2.1.1.2 Transitive moderation authority](#42112-transitive-moderation-authority)
    + [4.2.2 Authoring `post/role`](#422-authoring-postrole)
    + [4.2.3 Relevant roles](#423-relevant-roles)
    + [4.2.4 Declining roles](#424-declining-roles)
    + [4.2.5 Applying and resolving roles](#425-applying-and-resolving-roles)
      - [4.2.5.1 Precedence examples](#4251-precedence-examples)
        * [4.2.5.1.1 Local user rules](#42511-local-user-rules)
        * [4.2.5.1.2 The *relevant role* with the most capabilities trumps roles that have lower capabilities](#42512-the-relevant-role-with-the-most-capabilities-trumps-roles-that-have-lower-capabilities)
        * [4.2.5.1.3 The default role is a normal user](#42513-the-default-role-is-a-normal-user)
        * [4.2.5.1.4 Combined example](#42514-combined-example)
  * [4.3 Displaying posts](#43-displaying-posts)
  * [4.4 Moderation actions](#44-moderation-actions)
    + [4.4.1 Undoing a moderation action](#441-undoing-a-moderation-action)
      - [4.4.1.1 Undoing `post/block` example](#4411-undoing-postblock-example)
    + [4.4.2 Relevant moderation actions](#442-relevant-moderation-actions)
    + [4.4.3 Applicable moderation actions](#443-applicable-moderation-actions)
    + [4.4.4 Applying moderation actions](#444-applying-moderation-actions)
    + [4.4.5 Conflicting moderation actions](#445-conflicting-moderation-actions)
    + [4.4.6 Dropping posts](#446-dropping-posts)
      - [4.4.6.1 Undoing the drop action](#4461-undoing-the-drop-action)
  * [4.5 Post privacy](#45-post-privacy)
    + [4.5.1 Undoing a local-only post](#451-undoing-a-local-only-post)
    + [4.5.2 Authenticated encryption](#452-authenticated-encryption)
  * [4.6 Network blocks](#46-network-blocks)
    + [4.6.1 Impact on synchronization](#461-impact-on-synchronization)
      - [4.6.1.1 Authenticated Connections](#4611-authenticated-connections)
  * [4.7 Moderation seed](#47-moderation-seed)
    + [4.7.1 Problem scenario](#471-problem-scenario)
    + [4.7.2 Solution](#472-solution)
    + [4.7.3 Moderation seed example](#473-moderation-seed-example)
- [5 Wire formats](#5-wire-formats)
  * [5.1 Posts](#51-posts)
    + [5.1.1 Header](#511-header)
    + [5.1.2 `post/role`](#512-postrole)
      - [5.1.2.1 `role = 2 set normal user`](#5121-role--2-set-normal-user)
      - [5.1.2.2 `role = 1 set mod`](#5122-role--1-set-mod)
      - [5.1.2.3 `role = 0 set admin`](#5123-role--0-set-admin)
    + [5.1.3 `post/moderation`](#513-postmoderation)
      - [5.1.3.1 Acting on users](#5131-acting-on-users)
      - [5.1.3.2 Acting on posts](#5132-acting-on-posts)
      - [5.1.3.3 Acting on channels](#5133-acting-on-channels)
      - [5.1.3.4 `action = 0 hide user`](#5134-action--0-hide-user)
      - [5.1.3.5 `action = 2 hide post`](#5135-action--2-hide-post)
      - [5.1.3.6 `action = 4 drop post`](#5136-action--4-drop-post)
      - [5.1.3.7 `action = 6 drop channel`](#5137-action--6-drop-channel)
    + [5.1.4 `post/block`](#514-postblock)
      - [5.1.4.1 Dropping a user](#5141-dropping-a-user)
    + [5.1.5 `post/unblock`](#515-postunblock)
  * [5.2 Message Formats](#52-message-formats)
    + [5.2.1 Requests](#521-requests)
      - [5.2.1.1 Moderation State Request](#5211-moderation-state-request)
- [6 Security Considerations](#6-security-considerations)
  * [6.1 Out of scope Threats](#61-out-of-scope-threats)
  * [6.2 In-scope Threats](#62-in-scope-threats)
  * [6.3 Susceptibilities](#63-susceptibilities)
    + [6.3.1 Privilege Escalation](#631-privilege-escalation)
    + [6.3.2 Message Omission](#632-message-omission)
  * [6.4 Protections](#64-protections)
    + [6.4.1 Inappropriate Use](#641-inappropriate-use)
    + [6.4.2 Denial of Service](#642-denial-of-service)
    + [6.4.3 Attacker Expulsion](#643-attacker-expulsion)
    + [6.4.4 Spoofing](#644-spoofing)
- [7 Normative References](#7-normative-references)
- [8 Informative References](#8-informative-references)

## 1 Introduction

Cable is a peer-to-peer protocol facilitating group chat. As an approach
to group chat it is unique in that no one user has complete authority
over a particular chatroom; the chat lacks owners.

This ownerless property yields benefits, such as the chatroom able to
exist over spans of time outlasting the original set of
initiative-takers, while also opening up questions, such as how content
moderation may work in such a setting. This document strives to provide
an answer for that question.

### 1.1 Background

*This section is non-normative*

Historically, group chat has always implied a relationship to governance
and to content moderation. Whether to knowingly or not take a stance on
explicitly shirking responsibility for the contents of a chatroom or
deciding to introduce a hierarchical system of users such that some
users may censor and remove others.

Cabal, the existing distributed peer-to-peer computer software which
laid the groundwork for creating cable, implemented a subjective
moderation system in which all users were empowered with the action to
hide, from their own view of the chat, users found to be disruptive.
More significantly, each user was also able to designate roles for
others users such that the actions taken by the target users were also
applied for the designating user. The described approach existed only as
a set of modules and scattered references in documentation, prompting
the creation of this specification which goes further in terms of its
operations.

### 1.2 Overview

*This section is non-normative*

The system described in this document details an approach for content
moderation tailored to cable's properties as a protocol. The moderation
system adds two different concepts: roles and moderation actions. The
set of roles—administrators, moderators, and normal users—grant
different capabilities within the moderation system in regards to
setting roles on other users and performing moderation actions.

Moderation actions revolve around various means of hiding and removing
chat messages (with a specific focus towards `post/text`), as well as
users and channels. The types of actions that can be taken are to *hide*
(meaning to remove from view but keep in storage), to *drop* (to remove
from storage and from view), and to block (only in respect to users: to
inhibit post synchronization between the blocked user and the blocking
user).

The described system steps away from imposing an objective hierarchy and
instead introduces a notion of subjectivity. Each user is granted with
maximal agency in terms of decision-making ability for that which
affects their view of the group chat and what messages are stored on
their device, and agency to delegate these capabilities to hide, drop
and to block on their behalf. We use the phrasing of considering the
perspective of *a local user*: the perspective of a particular user
running a client. Consequently, each user may have a different set of
administrators and moderators. The system puts an emphasis on the
actions of individual users, and empowers them, while also making
possible collective action through enabling users to delegate moderation
authority to trusted users.

### 1.3 Differences between actions, roles, moderators, and administrators

*This section is non-normative*

We now briefly mention core characteristics of how and when actions and
roles are applied, what happens during role revocation, and sketch a
mental model for understanding role application.

#### 1.3.1 Roles

*This section is non-normative*

Moderators are effectively users delegated future moderation
responsibility but who lack transitivity i.e. roles are not applied when
issued from a moderator. Administrators are users delegated with
moderation as well as with role assignment responsibility. They enable
transitivity, applying moderation actions outside of a user's directly
issued roles, whereas moderators do not.

Roles issued from an administrator only start applying from the point in
time they received the role, meaning their historically issued roles are
not inherited. If an administrator has their role revoked all their
issued roles are also revoked and cease applying, reverting effects to
the extent possible.

#### 1.3.2 Roles as vouching

*This section is non-normative*

The set of active moderators and administrators for a user at any point
in time is determined by what can be described as a vouching system.
Instead of administrators' role assignments competing, the latest issued
role for a particular user winning (i.e. being applied), a given user's
role is instead determined by the *highest* role (the role with the most
capabilities) issued for them. The metaphor of vouching is used because
that user retains their role so long as they have others "vouching" for
them (maintaining their issued role for the user).

This vouching applies within the constraints of the overall moderation
system: being a *subjective* system means outcomes depend on a
particular user's point of view and their set of
administrators—"vouches" from users not regarded as administrators do
not apply, and "vouches" from an administrator who lost their role no
longer apply, either. A local user may also, at any time, override
decisions made by delegated users through themselves issuing a role or
moderation action.

#### 1.3.3 Actions and revocation

*This section is non-normative*

Moderation actions are applied from the point in time a user was
delegated moderation authority, i.e. past moderation actions are not
inherited. If a user with moderation authority has their role revoked
the actions they issued should remain applied.

That is, when role revocation occurs moderation actions continue to be
applied whereas role assignments do not.

### 1.4 Design philosophy

*This section is non-normative*

Tradeoffs have been made when designing the described system, choosing
in places to sacrifice expressivity of actions and roles in exchange for
relative simplicity and flexibility in terms of implementation. Emphasis
has also been put on individual user privacy, such that moderation
actions may be privately issued, as well as a facility for users to
renounce and deny all roles applied to them by others.

## 2 Conformance requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [BCP
14](https://www.rfc-editor.org/bcp/bcp14), [RFC
2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC
8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

### 2.1 Terminology and other Conventions

This document makes use of the definitions and terminology defined in
the Cable Wire Protocol document, in particular section 3.
[Definitions](https://github.com/cabal-club/cable/blob/main/wire.md#3-definitions)
such as *client*, *cabal*, *peer*, *user*, *post*, *hash*, *channel*,
and so on.

## 3 Definitions

**keywords _before_, _after_, _newer_, _older_**: used to compare field
`timestamp` between two or more posts. A post whose `timestamp` is less
than the `timestamp` of another post happens _before_ and it may also
be referred to as the _older_ of the two posts, and vice versa.

**field `privacy`**: determines whether a post will be stored locally in
an encrypted format, never being sent as part of requests. Enables
actions taken privately by a user without other users able to know about
it.

**field `reason`**: an optional description for an action, useful for
understanding why it was taken.

**fields `recipient`, `recipients`**: when acting on users: the `public_key` the action
is being performed on; for posts: the hash of the post being acted on.

**moderation actions**: also referred to as *actions*. Actions on users,
channels, and posts issued via `post/block`, `post/unblock`, or
`post/moderation`.

**moderation authority**: users with role administrator or moderator.
Users with moderation authority may issue moderation.

**moderator**: also referred to as *mod*. Has moderation authority, may
not assign roles granting moderation authority.

**administrator**: also referred to as *admin*. Has moderation authority
and may assign roles mod or admin.

**moderation seed**: a set of public keys specifying users to regard as
moderation authorities at the time of joining a cabal.

**roles**: roles issued via `post/role`.

**local user**: a user possessing knowledge of the private key for an
ed25519 signing keypair and running a client.

**subjective moderation**: a system of content moderation where
moderation authority originates from the perspective of the local user.
A cabal's different users may have diverging views on which users have
moderation authority. Subjective moderation grants each local user the
maximum agency in terms of taking moderation actions and issuing roles.

**role transitivity**: moderation authority granted through transitive
admin role assignments.

## 4 Data model

### 4.1 Channel context

A user MAY have different roles in different channels and actions MAY
target different channels. Therefor we say that a post is issued for a
particular *channel context*. A channel context MUST be considered to be
either the entire cabal or a specific channel.

The channel context applies to `channel` fields found in `post/role` as
well as to `post/moderation` (such as when issued with
`action = 0 hide user`).

#### 4.1.1 The cabal context

When a post's channel context is set to the entire cabal the effect of
that post MUST be considered applicable to all channels comprising the
cabal.

### 4.2 Roles

Roles are represented by post type `post/role`. Given the subjective
nature of this moderation system, any user MAY author a `post/role` for
another user. However, only roles authored by users with role admin
SHOULD be applied. Application of roles is tackled in depth in section
[Applying and resolving roles](#425-applying-and-resolving-roles).

#### 4.2.1 Moderation authority

Users with moderation authority SHOULD have their moderation actions
take effect for those users who regard them as having moderation
authority.

Who is regarded as having moderation authority:

-   the local user
-   users with role admin
-   users with role moderator

##### 4.2.1.1 Subjective authority

All users have roles, *which* role a given user has depends on the
perspective of a particular user. Roles granting moderation authority
MUST be considered from the point of view of the local user.

###### 4.2.1.1.1 Role transitivity

When the local user authors a `post/role` that sets another user as role
admin it becomes possible for roles to be applied which the local user
has not issued themselves. We refer to the notion of roles being issued
by admins who may or may not have been issued their own role from the
local user as *role transitivity*.

If the local user regards another user as role admin then roles issued
from the admin SHOULD generally be applied. Section [Applying and
resolving roles](#425-applying-and-resolving-roles) fully details how
and when role assignments should be resolved and applied.

###### 4.2.1.1.2 Transitive moderation authority

If there exists no role transitivity from the local user to another
user, then the other user MUST be regarded to have no moderation
authority. If role transitivity exists, then the role on the other user
MUST determine whether they possess moderation authority.

#### 4.2.2 Authoring `post/role`

When authoring a `post/role` the following MUST apply:

1.  A single user MAY issue roles for many users.
2.  A user MUST have at most 1 issued role per combination of recipient
    and channel context.
3.  A role MUST be overridden if a new role is issued where post author,
    recipient, and channel context remain the same. The post author's
    newly issued role for the recipient SHOULD replace their previously
    issued role, and the post representing the older role SHOULD be
    regarded obsolete and MAY be discarded.

#### 4.2.3 Relevant roles

The *relevant roles* is the set of roles considered active. A particular
role is considered *active* if it is a user's latest authored role for a
given recipient and channel context.

A particular `post/role`, as authored by a single user, MUST be
considered a *relevant role* if:

-   The recipient accepts roles (their latest `post/info` sets
    `accept-role: 1`), and
-   the role has not been made obsolete by its author.

Example:

-   User Aleph authors a `post/role` setting `recipient = Bert.public_key` and
    `role = 1 mod` where the channel context is the entire cabal
-   Some time later, Aleph authors another `post/role`, setting
    `recipient = Bert.public_key` and `role = 2 admin` for the previous
    channel context

Result:

User Aleph views user Bert as an admin for the entire cabal, and Aleph's active
role for Bert is that `post/role` setting Bert as an admin (Aleph's newest role
for Bert). The role from Aleph to Bert setting Bert as a moderator is obsolete and
may be discarded.

#### 4.2.4 Declining roles

A user MAY opt-out of roles, such as moderator or admin, being assigned
them by other users by setting `post/info`'s key `accept-role` with
`value = 0`.

-   Set `accept-role = 1` for a user accepting roles being assigned to
    them.
-   Set `accept-role = 0` for a user opting-out of roles being assigned
    to them.

The default value is `accept-role = 1`.

A user with `accept-role = 0` set MUST be regarded as a normal user,
irrespective of any roles previously set on them. Posts of type
`post/role` that were previously issued for a recipient that has opted
out of roles SHOULD be discarded.

When authoring a `post/role`, field `recipient` MUST NOT contain a
`public_key` corresponding to a user with `accept-role = 0` set. A user
who has opted out MAY still author `post/role` for users accepting role
assignments.

#### 4.2.5 Applying and resolving roles

When the local user applies role admin for a user, any roles issued by
the admin from that point on MUST be applied. Roles issued from before
the admin assignment MUST NOT be applied; i.e. historic roles should not
be inherited.

If an admin has their role revoked, the roles they issued for that
channel context MUST NOT remain applied. If the role was revoked for the
cabal context then their issued roles MUST be revoked for all channels
except those where they still retain role admin.

If multiple active roles are set for a single user, for example as a
result of roles issued by the local user and other admins, the way to
resolve that user's role MUST be as follows, in order of precedence from
top to bottom:

1.  The local user's role is always admin
2.  The local user's issued roles trump all other roles
3.  The *relevant role* with the most capabilities trumps roles that
    have lower capabilities
4.  The default role is a normal user

If possible, roles authored by different users SHOULD be applied such
that the roles issued with an older timestamp are applied before roles
issued with a newer timestamp.

##### 4.2.5.1 Precedence examples

###### 4.2.5.1.1 Local user rules

Rules 1-2 state that the local user must be regarded an admin,
irrespective of roles issued for the local user. Additionally, the local
user's assigned roles should also be applied in preference to roles
originating from other users.

Consider the following example. The local user sets two other users as
role admin. One of the admins sets the other admin as a normal user,
removing their moderation authority. Given rule 2, the demoted admin
should still be regarded as an admin from the perspective of the local
user.

Another example: the local user sets user Aleph as an admin and sets another
user, Xu, as a normal user. Admin Aleph issues a `post/role` setting Xu as a
mod. Given rule 2, from the perspective of the local user, Xu should not
be promoted to mod and should instead remain a normal user. This can be
employed to ignore role assignments for a user whose moderation
decisions the local user knows upfront that they disagree with.

###### 4.2.5.1.2 The *relevant role* with the most capabilities trumps roles that have lower capabilities

Given users Ursula, Aleph, Bert, and Cashew from Ursula's perspective where:

-   Ursula sets Bert as an admin
-   Ursula sets Aleph as an admin
-   Aleph sets Cashew as a mod
-   Bert sets Cashew as an admin

Then Ursula should regard Cashew as having role admin, because role
admin (issued by Bert) has more capabilities than role mod (issued by
Aleph).

###### 4.2.5.1.3 The default role is a normal user

If a user does not have any role assigned to them, they MUST be regarded
as a normal user.

###### 4.2.5.1.4 Combined example

Using a combination of the previous precedence rules, we now consider
the following scenario involving users Ursula, Aleph, and Bert from
Ursula's perspective:

1.  Ursula sets Bert as an admin for the cabal
2.  Ursula sets Aleph as a mod for channel `test`
3.  Bert sets Aleph as an admin for the cabal
4.  Ursula sets Aleph as a normal user for the cabal

After applying step 3, Ursula considers Aleph to have role mod in channel
`test` (rule *2. the local user's roles trump all other roles*) and as
role admin in all the other channels of the cabal (rule *3. the relevant
role with the most capabilities trumps roles that have lower
capabilities*).

After step 4, Ursula considers Aleph to have role normal user
in the entire cabal with the exception of channel `test` where they
retain their moderator role (rule *3.  The relevant role with the most
capabilities trumps roles that have lower capabilities* causes mod to be
retained in `test` due to being a relevant role and having greater
capabilities).

### 4.3 Displaying posts

When we refer to "displaying a post", we mean that the contents encoded
by the post SHOULD be rendered in a user-facing client in some way
meaningful and interpretable by a user. In this document, we primarily
concern ourselves with displaying posts representing moderation actions
or roles.

A displayed post SHOULD include information about who authored it.
Information about the encoded action SHOULD be rendered in a human
meaningful format, such as a descriptive string rather than the varint
representing the action.

If a post has a field `reason` where `reason_size` \> 0, then the
`reason` string SHOULD be included in the display in some way. For
instance, if a post has been hidden by a `post/moderation` with
`action = 2 hide post` and the post has a populated `reason` field, then
the `reason` contents MAY be displayed instead of the hidden post.

For posts acting on multiple recipients, we RECOMMEND that display of
long lists of recipients be done in a compact way.

### 4.4 Moderation actions

A moderation action is any post of type `post/moderation`, `post/block`,
and `post/unblock`.

One user MAY have many moderation actions apply to them at once; for
instance, a user may be hidden and blocked. 

Actions for a given recipient taken on both the cabal context and on a
specific channel SHOULD interact in the following way: A user that is
e.g. hidden for the cabal context and subsequently unhidden in a
specific channel SHOULD be hidden in all channels except the channel
where they were unhidden.

When a moderation action is acted on, we say it *takes effect*. By take
effect, we mean that the acting client applies the encoded action:
hiding, blocking, or unblocking a user, dropping a post, and so on. For
instance, for a `post/moderation` with `action = 0 hide user` set, the
effect is that the `recipients` defined in the post should have their
posts of type `post/text`, both new and old, no longer be displayed. The
action takes effect for the author of the post and any users who regard
them as a moderation authority.

In general, moderation actions that take effect SHOULD be displayed in
some manner. Moderation actions that do not take effect, for instance
due to lack of moderation authority, MAY be displayed.


#### 4.4.1 Undoing a moderation action

In the rest of the text we will reference moderation actions being
"undone", or one moderation action undoing the effects of another. We
now describe what is meant.

When an "effect" is undone, we mean that the reverse occurs: if a user
was blocked, they are now unblocked. If a post was hidden, it is now
unhidden. If a channel was dropped, it is now undropped.

The mechanism of *undoing* a moderation action's encoded effect (e.g.
hiding a post, blocking a user) SHOULD be accomplished by authoring a
corresponding undoing action of which there is one for each moderation
action that can be taken.

We now consider the topic of undoing a moderation action by way of an
example.

##### 4.4.1.1 Undoing `post/block` example

User `issuer` authors a post of type `post/block` to block another user,
`blocked`, and sets `recipients = [blocked.public_key]`. When `issuer`
wants to "undo the block", they author a `post/unblock` for the
recipient that was previously blocked, `blocked.public_key`. In other
words: `post/unblock` undoes a `post/block`.

We can illustrate the example using the contents of the two posts,
`post/block` and `post/unblock`, presented below in JSON-like
pseudocode:

First, user `issuer` blocks user `blocked`:

    {
        public_key: issuer.public_key,
        signature,
        num_links,       
        links,
        post_type: 8,
        timestamp: 1600000000000
        reason_size: 0,
        reason: "",
        privacy: 0,
        recipient_count: 1,
        recipients: [blocked.public_key],
        notify: 1,
        drop: 0
    }

(Fields `signature`, `num_links`, `links` are left unspecified as they
are irrelevant for the example.)

Undoing the above `post/block` with a `post/unblock` would then look
like:

    {
        public_key: issuer.public_key,
        signature,
        num_links,       
        links,
        post_type: 9,
        timestamp: 1700000000000
        reason_size: 0,
        reason: "",
        privacy: 0,
        recipient_count: 1,
        recipients: [blocked.public_key],
        undrop: 0
    }

Note how `recipients` is the same for `post/block` and `post/unblock`. We
also see that `post/unblock` occurs with a timestamp greater than the
timestamp in the `post/block`.

#### 4.4.2 Relevant moderation actions

A moderation action is regarded as a *relevant action* if it has not
been undone by a newer action from the same post author.

If the effects of one moderation action are undone by another action,
the undoing action is the relevant action and the older action MUST be
considered obsolete and MAY be discarded.

Example:

-   User Aleph authors a `post/moderation` hiding user Bert in channel `test`
-   Some time later, Aleph authors a `post/moderation` unhiding Bert in channel
    `test`
-   The `post/moderation` unhiding Bert is (one of) Aleph's relevant moderation
    actions, in the channel context represented by channel `test`, for Bert

#### 4.4.3 Applicable moderation actions

A moderation action MUST be regarded as an *applicable action* if it is
considered a *relevant action* and if it was issued by a user who at the
time of issuing held moderation authority.

For a moderation action to be regarded applicable, the timestamp of the
moderation action MUST be newer than the timestamp of the `post/role`
causing the author to become a moderation authority.

A `post/role`'s field `recipient` MUST NOT contain the author's
`public_key`, i.e. roles targeting oneself are disallowed.

#### 4.4.4 Applying moderation actions

Only moderation actions regarded as applicable SHOULD be applied.

When one "applies an action", we mean that the effects encoded by the
moderation action take effect.

If undoing an action, the effects of the action SHOULD be undone to the
extent possible.

Users applying a moderation action issued by another user with
moderation authority MUST NOT issue a post identical to the moderation
action taking effect. Instead, they SHOULD apply the effects of the
moderation action without issuing any new post.

A moderation action MUST remain applied until another action undoes it.
If an author deletes their moderation action, its effects MUST be
undone.

Older moderation actions from before a user achieved moderation
authority MUST NOT be applied. The actions were issued during a time
when they lacked authority.

Moderation actions applied when a user had moderation authority at the
time of issuing but whose authority has been revoked MUST remain
applied. The actions were issued during a time in which they had
authority.

Moderation actions should be associated with their post author such that
actions may be transparently inspected and retracted if regarded as
unwarranted.

If possible, moderation actions SHOULD be applied in order of oldest to
newest. If, for instance, three moderation actions were newly received
with with the following timestamps: `1700000000000`, `1711111111111`,
`1722222222222`. Then the order of application SHOULD be:

1.  First apply the post with `timestamp: 1700000000000`,
2.  followed by `timestamp: 1711111111111`,
3.  and finally `timestamp: 1722222222222`.

#### 4.4.5 Conflicting moderation actions

If the effects of two moderation actions conflict, for example when
actions to hide a user *and* unhide a user are issued by two different
users with moderation authority for the same recipient and channel
context, resolve as follows:

1.  If one of the actions originate from the local user, the effects of
    the local user's action MUST trump those of the other action,
    irrespective of whether it is the newest of the two actions.
2.  Otherwise, the action with the latest timestamp SHOULD be the action
    to take effect.

Moderation actions affecting users with moderation authority SHOULD NOT
be applied unless originating from the local user. Instead, information
that the action was issued SHOULD be displayed. An option to apply the
action MAY be displayed.

Example:

-   User Ursula sets users Aleph and Bert to role mod
-   Aleph blocks Bert
-   Ursula should not apply Aleph's block of Bert; Aleph's action should be displayed
    for Ursula

A user MAY issue moderation authority for a user that has been hidden or
blocked. In that case, the role recipient SHOULD still remain hidden or
blocked until a corresponding undoing action has been issued.

#### 4.4.6 Dropping posts

A post may be dropped through certain moderation actions. Whenever a
post is referred to as being "dropped", the behaviour must be as
follows.

Dropped posts SHOULD always be removed from the local store, and a
dropped post SHOULD NOT be requested.

Dropping a post is semantically different from deleting a post. A delete
may not be undone, and may only be issued for self-authored posts.
Dropping, however, MAY affect posts authored by other users and MAY also
be undone.

When a post is dropped, the action initiating the drop SHOULD be
displayed and include information about the author of the dropped post
and the author of the dropping action.

Dropping a post SHOULD result in the hash of the dropped post being
associated with the hash of the dropping post. By tracking these hashes,
a client can prevent resynchronizing the dropped post. This also allows
undoing the action, if acting in a timely manner, in the event that a
user with moderation authority abuses their delegated power.

##### 4.4.6.1 Undoing the drop action

In addition to dropping a single post, a channel, and a user MAY also be
dropped. The result of both of these actions is described in the
sections for [`post/moderation`](#513-postmoderation) and
[`post/block`](#514-postblock), but briefly: dropping a channel drops
all of the posts associated with that channel. Dropping a user drops all
of the posts authored by that user.

But how does one undo the dropping action, making retrieval possible
again? This is done with the following undropping-focused actions:

-   For a dropped post, users may author a `post/moderation` with
    `action = 5 undrop post` and `recipients = hash(dropped_post)`.
-   For a dropped channel, users may author a `post/moderation` with
    `action = 7 undrop channel` and
    `channel = <name of dropped channel>`; `recipients` should be left
    empty when operating on a channel.
-   For a dropped user, users may instead author a `post/unblock`,
    setting `undrop = 1` and `recipients` set to the public key of the
    dropped user.

When undoing a drop, the hashes of the (previously) dropped posts MAY no
longer be tracked and MAY be requested with a `Post Request`.

### 4.5 Post privacy

If a post has a field `privacy` and it is set to `1 = local-only`, that
post MUST NOT leave the local database. The post, or its hash, MUST NOT
be sent in response to requests from other users. Consequently,
knowledge of such a post only concerns the local user.

To guard the contents of a post with `privacy = 1 local-only` against
unintentional sharing, the post binary MUST be encrypted using the
procedure outlined in section [Authenticated
Encryption](#452-authenticated-encryption). Implementations MUST only
store the encrypted post, referencing it with the hash of the
unencrypted post.

#### 4.5.1 Undoing a local-only post

Consider the following scenario from the perspective of the local user.
If one post undoes the effects of another post where the first post's
field `privacy` has been set as `privacy = 1 local-only` then the
undoing post MUST also have set its field `privacy` set to the same
value i.e. `privacy = 1 local-only`.

For example, say the local user issued a `post/moderation` setting
fields:

* `action = 0 hide user`, 
* `recipients = hidden-user.public_key`, and 
* `privacy = 1 local-only`

Call this *post 1*.

The local user may have decided to set field `privacy = 1`, for
instance, in seeking to avoid initiating a conflict by publicly hiding
`hidden-user`. Some time later, they may feel like unhiding `hidden-user`.
This is done by authoring a `post/moderation` with `action = 1 unhide
user` and including `hidden_user.publicKey` in field `recipients`.

Call this *post 2*. 

The result being that *post 2* undoes the effects of *post 1*. 

If the following conditions hold true:

* field `recipients` overlap,
* a newer post undoes the action encoded in an older post, 
* and the older post has `privacy = 1 local-only`

then the newer post (*post 2* in the example above) MUST also have its
field `privacy` set as `privacy 1 = local-only`.

#### 4.5.2 Authenticated encryption

The encryption step employs the local signing keypair and is known as
*authenticated encryption*. We use the *XSalsa20-Poly1305* variant.
Application of this encryption step will result in the encryption of a
single post using a key only known, and derivable, by the local user. We
now outline the general steps for performing the authenticated
encryption:

1.  Derive a X25519 keypair using the local user's ed25519 signing
    keypair; call the derived keypair the *encryption keypair*.
2.  Generate a 24 byte nonce using a cryptographically secure
    pseudorandom number generator
3.  Perform a key exchange with X25519, using only the encryption
    keypair for both sides of the exchange, to derive a symmetric key.
4.  Authenticate the payload along with the generated nonce using
    Poly1305, creating an authentication tag.
5.  Use the derived symmetric key to encrypt the post payload with
    XSalsa20.
6.  Store the nonce as well as ciphertext together with the
    authentication tag (commonly referred to as *combined mode*).

To decrypt, derive the symmetric key from the encryption keypair using
X25519. Separate the stored ciphertext, nonce, and authentication tag.
Decrypt the ciphertext with XSalsa20 using the symmetric key and the
stored nonce. Authenticate with Poly1305 using the plaintext, nonce, and
the authentication tag.

### 4.6 Network blocks

Network blocks affect post synchronization and are represented by post
type `post/block` and undone with `post/unblock`.

A user blocking another SHOULD NOT request nor store new posts from the
blocked user. In the event of receiving a blocked user's posts, the
received posts MUST be discarded. Peers SHOULD NOT forward a blocking
user's posts to a blocked user, the behaviour of which is described in
detail in section [Authenticated
Connections](#4611-authenticated-connections). The `post/block`
establishing the block SHOULD NOT be sent to the blocked user
`post/block` unless field `notify = 1` is set. Already stored posts
authored by the blocked user MAY continue to be stored and displayed
unless explicitly dropped by setting `drop = 1`.

It is important that block actions reach as many users as possible. If
the blocker shares at least one channel with another user that is not
the blocked user, then that user SHOULD receive the `post/block`.

#### 4.6.1 Impact on synchronization

Clients SHOULD associate each blocked user with whom is blocking them. We
illustrate this with the logical pairwise mapping of the blocking user
to the blocked user:
`[blocking-user.public_key, blocked-user.public_key]`.

In the following situations, a user acting as a terminal peer MUST
discard any posts received in a Post Response if they were authored by:

-   a user they have blocked, or
-   a user they have been blocked by

A user blocking another user through actions issued by moderation
authorities MUST behave as if they were the author of the block. A
blocked user MUST discard posts they receive from a user blocking them
if they have been notified of the block and if the author of the
received posts was the same as the author of the `post/block`.  There
is no requirement for a blocked user to keep track of the subjective
moderation graph of all other users in order to honor not storing
blockers.

The blocking semantics operate at their intended full capacity when
combined with the *Authenticated Connections* described in the next
section.

##### 4.6.1.1 Authenticated Connections

Through the Cable Handshake Protocol peers MAY establish authenticated
connections, causing each party of a successful connection to know the
ed25519 public key of the other.

The behaviour described in this section MUST apply to responses created
by the terminal peer as well as to forwarded responses and it MUST only
be applied for established authenticated connections.

If the local user establishes an authenticated connection with a peer
they block the connection SHOULD be terminated.

When a terminal peer receives a Post Request over an authenticated
connection where the requester's associated public key matches a user
known to be blocked (notably: in this case, the terminal peer MUST NOT
block the requesting user), the responder MUST omit any posts authored
by all users blocking the requester.

When a terminal peer receives a Post Request over an authenticated
connection where the requester's associated public key matches a user
known to currently block other users, the responder MUST omit all posts
authored by users the requester is blocking.

### 4.7 Moderation seed

In certain situations it may be useful for users to join a cabal with a
predefined set of users to regard as moderation authorities by default,
we refer to this as joining with a *moderation seed*.

#### 4.7.1 Problem scenario

Consider a cabal with a set of users who remain active posters in
commonly used channels despite having been hidden or blocked by most,
but not all, users. Users joining for the first time, who lack issued
moderation actions as well as users regarded as moderation authorities,
would be exposed to the posts of these lingering moderated users.

#### 4.7.2 Solution

To address the above scenario and others like it, users MAY join with a
*moderation seed*. A moderation seed describes a set of public keys and
roles to temporarily assign the users represented by those public keys.

The moderation seed for a set of recipients MUST be constructed in the
following manner: 

1. Take a recipient from the set in no particular order.
2. Write the bytes for the varint representing their moderation seed role assignment.
3. Immediately following the role bytes, write the bytes representing the
   recipient's public key.
4. Continue until the recipient set is empty.

Following the procedure for a non-empty set of recipients and their roles
results in a byte sequence consisting of `<varint role><32 byte ed25519 public key>` 
pairs, one pair for each user being assigned a role. 

The moderation seed MUST contain no more than 16 public key assignments.

Conceptually, the moderation seed MUST temporarily change the default
role of the users it references from that of a normal user to that of
the specified roles. Consequently, any user with role admin may assign
another role to one of the moderation seed referenced users just as they
would any other user. Implementations MUST NOT cause `post/role` posts
to be created as a result of applying the temporary roles bestowed by
the moderation seed.

Actions and roles issued by the moderation seed's referenced users MUST
be applied irrespective of when they were authored but with respect to
the capabilities of the roles they were assigned.

Users joining with a moderation seed MUST be notified of that and
instructions MAY be displayed on how to revoke it and undo its role
assignments. The moderation seed's referenced users SHOULD retain their
roles until assigned another role or until the moderation seed is
revoked by the local user. When a moderation seed is revoked, the public
keys it described MUST have their default roles return to that of a
normal user.

Actions and roles issued from moderation seed referenced users that have
been applied MUST remain applied even after revoking the moderation
seed.

Sharing of a moderation seed may occur in a similar out-of-band fashion
and in conjunction with transmitting the cabal key (e.g. other chat
programs, written on paper, etc). Clients may have other ways of
representing a moderation seed, such as a query string of a URI query
component, however implementations MUST support ingesting the described
moderation seed format. We illustrate the format further with the
example below.

#### 4.7.3 Moderation seed example

Three users Aleph, Bert, and Cashew (their respective public keys in hexadecimal
representation below) are assigned roles in a moderation seed. Users Aleph
and Bert are assigned admin (varint value `2`) and user Cashew is assigned role
mod (varint value `1`).

The users' ed25519 public keys (hexadecimal):

-   Aleph:
    `c869744624581c4a7dfd0452f1b70dd4289fd14245eeb0a0c2b3a87f0e3a5b9d`
-   Bert:
    `656f9b6195035a063dd1f1f50def3a5a6ee19005384c49e1740df7dc192f722f`
-   Cashew:
    `1f03bd1d7430e5d47cf197d0ec412707a7e211ee7d45f298bf596378dd4c14a4`

The resulting moderation seed, where each role varint is followed by the
public key assigned that role, would have the following hexadecimal
representation for a total of 99 bytes.

    ┌─role varint
    │┌──────────────────32 bytes ed25519 public key─────────────────┐
    2c869744624581c4a7dfd0452f1b70dd4289fd14245eeb0a0c2b3a87f0e3a5b9d
    2656f9b6195035a063dd1f1f50def3a5a6ee19005384c49e1740df7dc192f722f
    11f03bd1d7430e5d47cf197d0ec412707a7e211ee7d45f298bf596378dd4c14a4

In the above representation we have chosen the ordering of having Aleph's
role & public key being followed by Bert and finally by Cashew, but as stated
above the order has no significance so long as the sequence consists of
bytes conforming to the `<role varint><32 byte ed25519 public key>`
scheme.

## 5 Wire formats

The following sections employ the field tables for encoding binary
payloads documented in section [6.1 Field
Tables](https://github.com/cabal-club/cable/blob/main/wire.md#61-field-tables)
of the Cable Wire Protocol document.

### 5.1 Posts

All posts described by this document begin with the 6-field header from
[section
6.2.1](https://github.com/cabal-club/cable/blob/main/wire.md#621-header)
of the Cable Wire Protocol document. Additionally, we add a new header
specific for moderation actions described in section
[Header](#511-header) below, which follow immediately after the
6-field post header.

| `post_type` numeric id | common name       | description                                            |
|------------------------|-------------------|--------------------------------------------------------|
| 6                      | `post/role`       | assign other users moderation authority                |
| 7                      | `post/moderation` | moderate posts, channels, and users                    |
| 8                      | `post/block`      | block users, preventing them from receiving your posts |
| 9                      | `post/unblock`    | unblock users, re-establishing post synchronization    |

#### 5.1.1 Header

Every moderation action MUST begin with the 6-field header from [section
6.2.1](https://github.com/cabal-club/cable/blob/main/wire.md#621-header)
of the Cable Wire Protocol document, and those 6 fields MUST be followed
by the following 3-field header:

| field       | type              | desc                                                             |
|-------------|-------------------|------------------------------------------------------------------|
| reason_size | varint            | length of the reason, in bytes                                   |
| reason      | u8\[reason_size\] | optional text description for reason of moderation action (UTF-8)|
| privacy     | varint            | 0 = public, 1 = local-only                                       |

`reason` MAY be used to communicate the rationale behind an action or as
a reminder for why it was taken. `reason` MUST be a valid UTF-8 string,
between 0 and 128 codepoints. `reason` MAY be left empty in which case
`reason_size` MUST be set to `0`.

If a moderation action or role should not be synchronized with other
users, field `privacy` MUST be set to `1` and the post binary MUST be
encrypted in the manner described in section [Authenticated
Encryption](#452-authenticated-encryption). If `privacy` is set to `0`
the post MUST be synchronized as any other post and MUST NOT be
encrypted per *Authenticated Encryption*.

#### 5.1.2 `post/role`

Propose roles and grant moderation authority to users.

| field        | type               | desc                                                                      |
|--------------|--------------------|---------------------------------------------------------------------------|
| channel_size | varint             | length of the channel's name, in bytes, set to 0 to apply to entire cabal |
| channel      | u8\[channel_size\] | channel name (UTF-8)                                                      |
| recipient    | u8\[32\]           | public key of role recipient                                              |
| role         | varint             | 0 = set admin, 1 = set mod, 2 = set normal user                           |

`post_type` MUST be set to `6`.

All roles MUST be regarded as exclusive and can be considered in terms
of increasing capabilities, listed below from most capabilities to
least:

-   admin
-   moderator
-   normal user

Upper levels MUST incorporate all capabilities of lower levels. A user
without any explicitly assigned role SHOULD be regarded as a normal
user, unless a moderation seed is active and affecting that user's
default role.

If a local user mistakenly issues a role for a user, they MAY delete it
with a `post/delete`.

##### 5.1.2.1 `role = 2 set normal user`

This role SHOULD be regarded as overriding any previous role set on
`recipient`, if any, setting their capabilities to the default role of
normal user.

Similar to roles mod and admin, this role when set by the local user
MUST NOT be overridden by new roles set by users with moderation
authority.

##### 5.1.2.2 `role = 1 set mod`

Has moderation authority and may issue moderation actions. Moderation
actions authored by a user with role mod MUST be applied according to
the behaviour described in sections [Applying moderation
actions](#444-applying-moderation-actions) and [Conflicting moderation
actions](#445-conflicting-moderation-actions). A user regarded as having
role mod MUST NOT have any `post/role` they author applied.

##### 5.1.2.3 `role = 0 set admin`

Has moderation authority and MAY issue moderation actions and MAY
propose roles:

-   `role = 0 set admin`
-   `role = 1 set mod`
-   `role = 2 set normal user`

#### 5.1.3 `post/moderation`

Perform moderation actions on users, posts, and channels.

| field           | type                        | desc                                                                                                      |
|-----------------|-----------------------------|-----------------------------------------------------------------------------------------------------------|
| channel_size    | varint                      | length of the channel's name, in bytes, set to 0 to apply to entire cabal                                 |
| channel         | u8\[channel_size\]          | channel name (UTF-8)                                                                                      |
| recipient_count | varint                      | set to 0 if acting on a channel and not a user or a post                                                  |
| recipients      | u8\[recipient_count \* 32\] | when operating on users contains the public key of a recipient and when operating on posts, the post hash |
| action          | varint                      | 0 = hide user, 2 = hide post, 4 = drop post, 6 = drop (archive) channel                                   |
|                 |                             | 1 = unhide user, 3 = unhide post, 5 = undrop post, 7 = undrop (archive) channel                           |

`post_type` MUST be set to `7`.

When acting on users or posts, `recipient_count` SHOULD be set to a
value between 1 and 16 otherwise it should be set to 0.

##### 5.1.3.1 Acting on users

Field `recipients` MUST be set to a sequence of 32 byte ed25519 public
keys representing the users being acted on.

If `channel_size` and `channel` are set when acting on a user, the
action MUST only apply to the specified channel. When actions apply to
the entire cabal, `channel_size = 0` MUST be set.

**Note**: dropping a user may instead be accomplished by authoring a
`post/block` and setting fields `drop = 1` and `privacy = 1`, see
section [Dropping a user](#5141-dropping-a-user).

##### 5.1.3.2 Acting on posts

Field `recipient_count` MUST be set to the number of post hashes being
acted on. Field `recipients` MUST be set to the post hashes.

Field `channel` MUST be set to the same value as the `channel` field of 
the posts being acted on.

##### 5.1.3.3 Acting on channels

Field `recipient_count` MUST be set to 0. Field `channel` MUST be set to
the channel name being acted on.

##### 5.1.3.4 `action = 0 hide user`

A hidden user's posts of type `post/text` MUST NOT be displayed. This
applies to old and new posts. Their posts MUST be stored and their new
posts MUST be requested. A hidden user's `post/info` MAY be displayed
differently for key `name`, such as displaying "hidden user" instead of
the value corresponding to key `name`.

The undoing action is `action = 1 unhide user`, which SHOULD cause the
user's posts to be displayed.

##### 5.1.3.5 `action = 2 hide post`

Hidden posts MUST NOT be displayed. The post being hidden MUST be of
post type `post/text`. All other post types MUST NOT be hidden with this
action. Hidden posts MUST still be stored and returned in response to
requests by other users.

If `reason` is set, it MAY be used to replace the contents of the post
with a content warning reflecting the contents of `reason`.

The undoing action is `action = 3 unhide post`, which SHOULD enable the
post to be displayed.

##### 5.1.3.6 `action = 4 drop post`

The post being dropped MUST be of either post type `post/text` or
`post/topic`. All other post types MUST NOT be dropped with this action.

Dropped posts MUST be removed from the local store in the manner
described in section [Dropping posts](#446-dropping-posts).

The undoing action is `action = 5 undrop post`. An undropped post SHOULD
be possible to request and store.

##### 5.1.3.7 `action = 6 drop channel`

Dropped channels MUST have all posts associated with that channel
dropped, regardless of post type. New posts authored in that channel
MUST NOT be requested nor stored.

Posts associated with a dropped channel MUST be removed from the local
store in the manner described in section [Dropping
posts](#446-dropping-posts).

`channel` MUST be set to channel name being dropped.

`recipient_count` MUST be set to `0`.

Dropped channels MUST NOT be represented in a `channel list response`.

The undoing action is `action = 7 undrop channel`. An undropped channel
MUST function like any other channel and posts authored for that channel
MUST be possible to request and store.

#### 5.1.4 `post/block`

Block users, affecting post synchronization between blocker and post
author.

| field           | type                        | desc                                                     |
|-----------------|-----------------------------|----------------------------------------------------------|
| recipient_count | varint                      | number of recipients to block                            |
| recipients      | u8\[recipient_count \* 32\] | public key of recipients, concatenated together          |
| drop            | varint                      | 0 = keep old posts, 1 = drop all old posts               |
| notify          | varint                      | 0 = do not notify blocked user, 1 = notify blocked user  |

`post_type` MUST be set to `8`.

`recipient_count` MUST be set to a value between 1 and 16. `recipients`
MUST contain a sequence of 32 byte ed25519 public keys.

Setting `drop` to 1 MUST drop all posts authored by `recipients` from the
local database. Setting `drop` to 0 MUST keep the posts in the local
database. Allowing posts to be kept enables situations where a user may
want to keep posts from blocked peers, for instance when preserving
evidence of abuse.

When `notify` is set to `0 = do not notify blocked user` the
`post/block` MUST NOT be transmitted to the blocked user.

If field `notify` is set to `1 = notify blocked user` the `post/block`
MUST be transmitted to the blocked user. Posts authored after the
`post/block` should function as otherwise specified and MUST NOT be
transmitted to the blocked user.

##### 5.1.4.1 Dropping a user

Users MAY "drop a user" without impacting post synchronization of that
user's posts for other users by setting `drop = 1` and `privacy = 1`.
The outcome of the operation MUST be that the recipient's posts are
dropped for the local user, who MUST NOT store nor request new posts
authored by the blocked user. Due to setting `privacy = 1`, other peers
SHOULD still store, receive, and synchronize the dropped user's posts.

#### 5.1.5 `post/unblock`

Unblock users, allowing the unblocked user's posts to be synchronized.

| field           | type                        | desc                                                    |
|-----------------|-----------------------------|---------------------------------------------------------|
| recipient_count | varint                      | number of recipients to unblock                         |
| recipients      | u8\[recipient_count \* 32\] | public key of recipients, concatenated together         |
| undrop          | varint                      | 0 = keep user's old posts as dropped, 1 = undrop posts  |

`post_type` MUST be set to `9`.

`recipient_count` MUST be set to a value between 1 and 16.

Setting `undrop` to 1 SHOULD undo the drop of all posts authored by
`recipients`, making them possible to retrieve again. Setting `undrop` to
0 SHOULD keep the old posts as dropped.

### 5.2 Message Formats

#### 5.2.1 Requests

Requests described in this document—as of writing there is only one
request—have the same message header as documented in
section [6.3.1 Message
Header](https://github.com/cabal-club/cable/blob/main/wire.md#631-message-header)
of the Cable Wire Protocol document.

##### 5.2.1.1 Moderation State Request

Request post hashes describing the latest moderation state for a set of
channels, and optionally subscribe to future state changes.

| field         | type                | desc                                                  |
|---------------|---------------------|-------------------------------------------------------|
| channel0_size | varint              | length in bytes of the 0th channel name, in bytes     |
| channel0      | u8\[channel0_size\] | the 0th channel name (UTF-8)                          |
| ...           |                     |                                                       |
| channelN_size | varint              | length in bytes of the Nth channel name, in bytes     |
| channelN      | u8\[channelN_size\] | the Nth channel name (UTF-8)                          |
| future        | varint              | whether to include future state hashes                |
| oldest        | varint              | milliseconds since UNIX Epoch. Set to 0 for all posts |

`msg_type` MUST be set to 8.

A request MUST indicate it is done specifying channels by setting the final
`channelN_size` to 0.

Responders SHOULD be a member of at least one of the requested channels
and respond with hashes for:

-   all `post/block` and `post/unblock`, regardless of post author,
    recipient, `oldest`, and channel
-   the set of *relevant roles* matching at least one of the requested
    channels or where the role is set for the entire cabal
-   the *relevant moderation actions* issued for one of the requested channels or
    those actions applying to the entire cabal

Responders MUST NOT include post hashes corresponding to `post/role` set
on users who have opted out of roles by setting `accept-role = 0` in
`post/info`.

`future` MUST be set to either `1` or `0`.

If `future = 1`, the responder SHOULD respond with future moderation state changes
as they become known to the responder. The request SHOULD be held open
indefinitely on both the requester and responder side until a Cancel Request is
issued by the requester, or the responder elects to end the request by sending
a Hash Response with `hash_count = 0`.

If `future = 0`, only presently known moderation state hashes SHOULD be included
in the response, limited by `oldest` and channel membership, and the request
MUST NOT be held open.

Field `oldest` is a time, expressed in milliseconds since UNIX Epoch, limiting
the amount of history sent when set. If `oldest > 0` then the posts whose hashes
are being returned SHOULD fulfill `post.timestamp >= oldest`. If `oldest = 0`
then this SHOULD be regarded as the limit being unset.

Implementations are RECOMMENDED to set a large value for `oldest`, for example
on the order of 1 year from the time of requesting: `oldest = <current time ms>
- 31536000000`.

A response to this request SHOULD return **all** known hashes concerning
types `post/block` and `post/unblock`, i.e. disregarding the value of field
`oldest` if set. This to ensure user safety by respecting issued
blocks such that they will reach all users of a channel.

## 6 Security Considerations

This section builds on section [7 Security
Considerations](https://github.com/cabal-club/cable/blob/main/wire.md#7-security-considerations)
from the Cable Wire Protocol document. We restate the relevant portions
from sections [7.1 Out of scope
Threats](https://github.com/cabal-club/cable/blob/main/wire.md#71-out-of-scope-threats)
and [7.2 In-scope
Threats](https://github.com/cabal-club/cable/blob/main/wire.md#72-in-scope-threats)
of that document and touch on how the attacks described in sections
[7.2.1
Susceptibilities](https://github.com/cabal-club/cable/blob/main/wire.md#721-susceptibilities)
and [7.2.2
Protections](https://github.com/cabal-club/cable/blob/main/wire.md#722-protections)
are affected by the introduction of the moderation system described in
this document.

### 6.1 Out of scope Threats

1.  Attacks on the transport layer by non-members of the cabal. This
    covers the *confidentiality* of a connection between peers, and
    prevent *eavesdropping* and *man-in-the-middle attacks*, as well as
    message reading/deleting/modifying.

2.  Attacks that attempt to gain illicit entry to a cabal, by posing as
    a member (*peer entity authentication*).

3.  Actors capable of deriving an Ed25519 private key from the public
    key via brute force, which would break data integrity and permit
    data insertion/deletion/modification of the compromised user's
    posts.

4.  Attacks that stem from how cable data ends up being stored locally.
    For example, an attacker with access to the user's machine being
    able to access their stored private key or chat history on disk.

### 6.2 In-scope Threats

Documented here are attacks that can come from *within* a cabal — by
those who are technically legitimate members and can peer freely with
other members.

### 6.3 Susceptibilities

#### 6.3.1 Privilege Escalation

The moderation system introduces roles and capabilities associated with
those roles concerning delegated message omission and impact on post
synchronization. However, the system's subjective nature limits the
amount of damage an attacker could do if managing to mount a privilege
escalation attack:

-   An attacker with a temporarily escalated role would have a limited
    effect and not necessarily affect all users of a cabal.
-   The temporarily escalated role could also be reversed by any user
    from themselves and those who view them as a moderation authority.
-   Acts committed by an attacker would be surfaced to affected users in
    the form of displayed posts corresponding to the roles and actions
    taken by the attacker, enabling tracing and reversal of the actions.
-   Any damage done by issuing moderation actions can be reversed by any
    user through issuing a corresponding undoing action. The same rings
    true for attacker-issued roles.

Attacks are furthermore limited by the fact that, generally, an elevated
role is normally only achieved through user-taken actions such as
joining with a moderation seed or setting a role for a particular user.

#### 6.3.2 Message Omission

The moderation system intentionally enables a multitude of
user-initiated options regarding message omission. All these
user-initiated actions leave a trace in the form of authoring a
corresponding post. A user may however enact message omission without
leaving any traces by following the procedure outlined in section [Post
privacy](#45-post-privacy), performing local data removal rendering the
act readable only by the local user.

### 6.4 Protections

#### 6.4.1 Inappropriate Use

Limited protection against INAPPROPRIATE USE is introduced by the
moderation system in that attackers may now have their post
synchronization capabilities impacted through being blocked by users as
well has having inappropriate posts selectively dropped by users. In
aggregate, where many users block an attacker as well as through blocks
issued by moderation authorities subscribed to by many users, the
attacker's effect on the cabal as a whole will be limited and in the
case of users for whom the attacker is blocked the lingering effects of
an attack are able to be erased through the facility of dropping all
posts authored by a user.

In particular, illegal number and similar attacks are handled by the
system in that individual posts and all posts by a single user can be
designated as dropped, causing them to be removed locally and prevent
resynchronization of the designated posts. This functionality enables all
users, individually and collectively, to avoid permanently compromising
their systems.

#### 6.4.2 Denial of Service

Cable's ability to deal with various DENIAL OF SERVICE attacks sees a
limited protection with the addition of the moderation system's ability
to drop entire users and all posts they have created, as well as
dropping entire channels and all posts associated with them.

Furthermore, peers detected to be disruptive and engaging in attacks may
be cut off from future connections through being blocked.

The block functionality combined with the Cable Handshake Protocol could
see the deployment of attack-resistant modes of operation: when an
attack is mounted and subsequently detected, clients could be switched
to a mode where the attacker is blocked and future connections from
unknown public keys are rejected until the mode is turned off.

Other aspects of DENIAL OF SERVICE attacks covered in the Cable Wire
Protocol document remain unaffected.

#### 6.4.3 Attacker Expulsion

Limited ability for ATTACKER EXPULSION has been introduced with the
moderation system's blocking functionality. Full expulsion, rending the
operator behind an attack from being unable to mount future attacks,
require capabilities not yet present for Cable, for instance that of
moving to a different cabal key.

#### 6.4.4 Spoofing

Limited protection against SPOOFING has been introduced with the
moderation system's ability to hide users and to label the action,
enabling users to hide spoofers with a reason field explicitly labeling
them as a spoofer.

## 7 Normative References

1.   [Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key
    Words", RFC 8174, May 2017](https://www.rfc-editor.org/rfc/rfc8174)
2.  [Bradner, S., "Key words for use in RFCs to Indicate Requirement
    Levels", RFC 2119, July
    2003](https://www.rfc-editor.org/rfc/rfc2119)
3.  [Rescorla, E. and B. Korver, "Guidelines for Writing RFC Text on
    Security Considerations", BCP 72, RFC 3552, July
    2003.](https://rfc-editor.org/bcp/bcp72)

## 8 Informative References

1.  [Cable Wire
    Protocol](https://github.com/cabal-club/cable/blob/main/wire.md)
2.  [Cable Handshake
    Protocol](https://github.com/cabal-club/cable/blob/main/handshake.md)
3.  [TrustNet](http://lup.lub.lu.se/student-papers/record/9020253)
4.  [cabal-core.js:
    Moderation](https://github.com/cabal-club/cabal-core/tree/6fe78849aaf7122250f53ed5df2025fe49f1fc11#moderation)
5.  [Webber, Christopher Lemmer, et al. "ActivityPub." W3C
    (2018).](https://www.w3.org/TR/activitypub/)
