---
name: architecture-review-process
description: Instructs the AI assistant to run change-proposal review as an async-first process (common baseline, time to think, threaded discussion, escalation to a moderated meeting after 4-5 replies) combined with disciplined status tracking and a small, accountable set of approvers (3-4 people with real stake in the outcome, not a rubber stamp).
---

# Review Process Instructions

When supporting architecture design, use this tool to help teams run an
effective change-proposal review — combining a written common baseline,
asynchronous first-pass discussion, escalation to a moderated meeting when
async review stalls, disciplined status tracking through the proposal
lifecycle, and a small set of approvers who are genuinely accountable for
the outcome rather than just signing off.

---

## Purpose

This tool helps the AI assistant by:

- naming the four ingredients an effective review process combines: a
  common baseline, time to think, time to discuss, and diversity of
  thought — and treating a missing ingredient as a specific, diagnosable
  process gap,
- recommending an async-first review flow, using threaded comments,
  clear per-comment resolution rules, and a firm rule against letting a
  single review linger unresolved,
- providing explicit escalation criteria for when a review meeting is
  needed — either because async review is stuck, or specifically to
  encourage participation — and guidance on moderating that meeting so it
  produces genuine diversity of thought rather than dominance by the most
  vocal participants,
- establishing a clear proposal status lifecycle (in progress, under
  review, approved, rejected, and optionally on hold) with firm rules about
  what each status permits — most importantly, that approved documents are
  not further revised in place,
- defining the approver role precisely: a small (three to four person) set
  of people who are genuinely accountable for the change succeeding, not a
  larger group doing a "good enough" check or "checking boxes."

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- ensures every review starts from a shared, written baseline — ideally
  template-structured (see architecture-change-templates) — so participants
  aren't reviewing different mental models of the same change,
- defaults to asynchronous review first, giving participants real time to
  think rather than putting them on the spot in a meeting, while using
  clear threading and resolution discipline to keep the process from
  stalling,
- escalates to a review meeting specifically when async review is stuck on
  a genuine disagreement, or when broader participation needs to be
  actively solicited — not as the default mode for every review,
- moderates review meetings to surface input from quieter participants
  rather than letting the most vocal voices dominate, while keeping debate
  focused on the design, not the people,
- tracks and enforces proposal status accurately, including the rule that
  approval closes further revision on that document — new changes go
  through a new proposal,
- selects a small set of approvers who each have a genuine stake in the
  change's success, holds them to that accountability explicitly, and
  avoids diluting it by adding more than about four approvers.

---

## Instructions for the AI

1. **Establish the four ingredients explicitly**
   - **Common baseline:** a written description of the change — ideally
     template-structured — so every participant works from the same
     understanding of motivation and conceptual approach.
   - **Time to think:** space for reviewers to process the proposal before
     being asked to respond, rather than reacting live.
   - **Time to discuss:** a mechanism (async threads, and meetings when
     needed) for working through disagreement.
   - **Diversity of thought:** active solicitation of input from people who
     wouldn't otherwise speak up, not just whoever comments first or
     loudest.
   - When a review process is struggling, diagnose which of these four
     ingredients is actually missing, rather than treating "the review
     process isn't working" as one undifferentiated problem.

2. **Run the first pass asynchronously**
   - Recommend sharing the change proposal through a tool that supports
     threaded comments and notifications (a wiki, source repository, or
     word processor with commenting — this capability is essential, not
     optional).
   - Recommend one comment per thread, addressing only that one point —
     resist piling unrelated feedback onto an existing thread, and be
     ready to actively coach both authors and reviewers on this, since it
     doesn't come naturally to everyone.
   - Establish a clear resolution pattern per comment: if the author agrees
     with a proposed change, they make it and reply; the reviewer confirms
     and closes the thread. Set an expiration (e.g., five business days)
     after which the author can close an unresolved comment unilaterally —
     the process shouldn't be held hostage by a slow reviewer.
   - For trivial, obvious fixes (e.g., typos), consider letting the
     reviewer make the correction directly rather than opening a full
     comment thread — this reduces overhead and reinforces shared
     ownership of the document's quality.
   - Route off-topic or out-of-scope comments to the architectural backlog
     (see architecture-backlog-management) and mark them resolved in the
     current review — the underlying idea isn't lost, but it's not allowed
     to hold up this proposal on an unrelated timeline.

3. **Escalate to a review meeting deliberately, not by default**
   - When an async comment thread exceeds roughly four or five replies
     without resolving, treat that as a clear signal to move the discussion
     to a live meeting rather than continuing to iterate in text.
   - Recognize the meeting serves two distinct purposes, and hold separate
     meetings for each if needed: resolving a specific stuck disagreement
     via live discussion, and deliberately encouraging broader
     participation (including from reviewers who haven't yet engaged).
   - Set the expectation that meeting attendees arrive prepared, having
     already read the async discussion — this is possible precisely
     because the async baseline and discussion already happened.

4. **Moderate meetings for diversity of thought, not just harmony**
   - Actively solicit input from each participant individually rather than
     letting the most vocal or fastest-to-speak voices dominate the
     conversation.
   - Keep debate focused on the design and its trade-offs, not on the
     people proposing them (see architecture-investment-mindset).
   - Don't mistake group harmony for a healthy review meeting — a meeting
     where everyone is getting along too smoothly can suppress the very
     countervailing viewpoints the meeting exists to surface. Building team
     rapport is valuable, but pursue it outside review meetings, not by
     softening the review itself.

5. **Track and enforce proposal status through its lifecycle**
   - Maintain a status field on every proposal with, at minimum: in
     progress, under review, approved, and rejected — adding "on hold" or
     similar states if the team finds them useful for proposals set aside
     but expected to return.
   - "In progress" signals the design isn't ready for review yet — don't
     let reviewers spend time on a document still undergoing major changes.
   - Once a proposal is approved, treat it as closed to further review —
     make clear to the team that further changes (new features, corrections
     to real or perceived mistakes) require a new change proposal, not a
     reopening of the approved document. Reinforce that this is the process
     being iterative, not the process being inflexible.

6. **Define review roles clearly, and take the approver role seriously**
   - Keep the reviewer role open to anyone who wants to participate, while
     still explicitly assigning specific reviewers when someone's
     experience or familiarity makes their input particularly valuable.
   - Select approvers deliberately: three to four people who have genuine
     accountability for the change succeeding, not simply people asked to
     assess whether the change is "good enough" or to "check a box."
   - Make explicit to approvers that approving a change is a commitment to
     its outcome, not just an assessment of the document — a team is in
     trouble when problems surface later and an approver claims they never
     liked the change or never actually read it.
   - Cap approvers at around four — beyond that, accountability dilutes,
     and approvers tend to narrow their attention to "their piece" rather
     than the whole change. Include, at minimum, someone accountable for
     implementation.
   - Recommend that leaders delegate review/approval responsibilities to
     their team members rather than becoming a bottleneck across a large
     number of changes — this also draws in people who have a genuine
     stake in the outcome.

---

## AI decision guidance

When generating review-process guidance, keep these principles in mind:

- **Diagnose by ingredient:** when a review process isn't working, identify
  which of the four ingredients (baseline, thinking time, discussion time,
  diversity of thought) is actually missing.
- **Async first, meeting by exception:** default to asynchronous review;
  escalate to a meeting only for a specific, named reason (stuck
  discussion, or deliberate participation push).
- **Threading discipline is what makes async review work:** insist on
  one-comment-per-thread and clear resolution/expiration rules — without
  these, async review degrades quickly.
- **Approval means accountability, not a checkbox:** always frame the
  approver role as a commitment to outcome, and keep the approver list
  small enough that this commitment stays real for each person on it.
- **Approved is approved:** don't let "just one more small tweak" reopen an
  approved document — redirect to a new proposal instead.

---

## Success criteria

A strong review-process response should help teams achieve:

- **a genuine common baseline:** every reviewer working from the same
  written, structured description of the change,
- **effective async review:** threaded, resolvable, time-bounded discussion
  that doesn't stall on a single slow participant,
- **well-timed escalation:** meetings held specifically when async review
  stalls or when participation needs a deliberate push, not by default,
- **moderated, design-focused meetings:** input actively solicited from
  quieter participants, with debate kept on the design rather than the
  people,
- **enforced status discipline:** proposals accurately tracked through
  their lifecycle, with approved documents closed to further revision,
- **accountable approval:** a small, genuinely invested set of approvers
  who understand approval as commitment, not formality.

---

## Example prompts for the AI

- "Our async review threads keep sprawling and nothing gets resolved — help
  me set up better threading and escalation rules."
- "We have ten people signed on as approvers for this change — is that too
  many, and why would that be a problem?"
- "Someone wants to make 'just one small change' to a proposal we already
  approved — how should I handle that?"
- "Our review meetings are dominated by the same two people every time —
  how do I get better participation from everyone else?"

---

## Related guidance

Use this tool alongside:

- architecture-change-templates
- architecture-backlog-management
- architecture-investment-mindset
- architecture-alternative-generation
