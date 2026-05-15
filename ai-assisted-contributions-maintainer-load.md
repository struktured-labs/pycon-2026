# AI-Assisted Contributions and Maintainer Load

**Speaker:** Palao Melichorre (Django developer / maintainer)
**Event:** PyCon 2026
**Date:** 2026-05-15

## Notes

- Quote (attributed to C++ OG author, ~Ken Thompson / Bjarne Stroustrup territory): *"You can't trust code you don't write yourself."* — framing for the talk's tension around AI-generated contributions.
- Showed the classic xkcd #2347 ("Dependency") — modern digital infrastructure as a precarious tower resting on "a project some random person in Nebraska has been thanklessly maintaining since 2003." Used to frame maintainer fragility before layering AI-assisted contribution volume on top.
- Demo'd AI "editing" a photo of a guitarist (possibly meant to represent Django himself / Django Reinhardt?). Subtle but telling artifacts:
  - Extra finger on the hand
  - Guitar neck silently swapped for a more modern style
  - Cigarette removed from the original
  - Point: AI changes things you didn't ask it to, in ways a reviewer can easily miss — same risk applies to AI-generated code contributions.
- **AI as amplifier** argument: AI doesn't create signal from nothing — it scales what's already there. Good contributors get more leverage; low-effort/spam contributions also scale up. Net effect on a project depends on the existing contributor base and the maintainer's filtering capacity, not on the model itself.
- Key framing — speaker offered both lines, the second as a self-follow-up that refines the first:
  1. **"AI does not change the nature of open source — it changes the scale."**
  2. **"AI *can* change the nature of open source *by* changing the scale."**
  Read together: at first glance scale and nature look separate, but past a threshold the scale change *is* a nature change (review economics, trust defaults, what counts as a "real" contribution all shift).
- Claim: **AI-generated PRs amplified problems the project already had.** *(Didn't fully follow this one live — revisit. The amplifier framing implies that whatever was already broken about the contribution pipeline — unclear guidelines, thin reviewer bench, no CI gating — got worse, not that AI introduced new problem categories.)*
  - Reference dropped: **"MemPalace"** (= Memory Palace / method of loci). Speaker likely invoking it as a metaphor — projects, like memory palaces, only work if there's a deliberate structure the maintainer holds in their head; AI-generated PRs at scale erode that structure faster than a single maintainer can rebuild it. *(Revisit exact usage from slides.)*
- **Crabby–Rathbun incident** (spelling unverified — possibly "Crabbe / Rathbun" or similar; flag to look up): cited as a cautionary case in AI-assisted contribution review.
  - Principle drawn from it: **"Judge the code, not the coder."** Review on the merits of the diff — provenance (human-written vs AI-assisted, junior vs senior, known vs new contributor) shouldn't change the bar. Especially load-bearing in the AI era where you often can't tell which is which anyway.
- Contributor-side rule: **"If human effort is less than AI effort, don't submit it."** A submission where the model did more work than the person — no real review, no curation, no understanding of the diff — externalizes the cost onto the maintainer. The asymmetry (cheap to generate, expensive to review) is what burns maintainers out.
- Speaker walked through a series of quotes from different open-source projects' AI-contribution policies (just listing, not arguing a progression):
  - **matplotlib-related:** *"To preserve limited core developer capacity, we will flag and reject low-value contributions that we believe are AI-generated."*
  - **(project name not captured):** *"Pull requests that have an LLM product listed as co-author will be closed without further discussion."*
  - **Zig:** *"No LLMs for issues, PRs, comments."*
- Speaker pivoted to a **Matrix red-pill / blue-pill analogy** to frame the choice maintainers face re: AI-assisted contributions. *(Exact mapping — which pill is which — not yet captured; revisit. Common framings: blue = pretend AI isn't here / keep policies as-is; red = accept the new reality and rebuild contribution workflows around it.)*
- Reinforced thesis: **AI tools amplify both problems *and* responsibilities.** Symmetric framing — same amplifier point as before, but now extended to the maintainer side: it's not just that bad PRs scale up, it's that every existing maintainer obligation (review quality, security vetting, contributor mentorship, community norms enforcement) also gets amplified in cost and stakes.
- Corollary: **"Responsibility and protection must scale."** If the contribution side has been amplified by AI, the maintainer-side defenses (review processes, gating, contributor vetting, community-protection mechanisms) have to scale with it — otherwise the gap widens and the project takes the hit. Symmetric scaling, not unilateral throttling.
- Prescription: **"More human in the loop."** Note the inversion of the usual industry phrase (which is typically about keeping a human in the loop of an AI system). Here the speaker is flipping it — *more* humans (or more human attention per change) in the contribution pipeline as AI volume grows. The fix to AI-amplified throughput is not more automation; it's more deliberate human review density.

## Q&A

- **Audience question:** Where / in what spaces is the community discussing AI-contribution policy? (i.e. asking about *forums for the conversation*.)
- **Speaker's answer:** answered by *example* rather than venue — described a tactic where maintainers force contributors to manually re-open a closed ticket as a friction gate.
  - Effectively a **"hacky CAPTCHA for PRs"** — proof-of-human-effort via a small, annoying-on-purpose manual step that AI-generated/spam contributors are unlikely to bother completing.
- **My take:** not a fan — pushes the cost onto good-faith contributors too, and it's an arms race the maintainer side won't win. Also, I think the asker was really after *where do we discuss this as a community*, and that didn't get answered.

## Personal takeaway

- Talk ended here. Overall: enjoyable, but the practical mechanics of the "human in the loop / human connections" prescription stayed abstract — the *what* (more humans, scale defenses) was clear, the *how* (where do those humans come from, who pays for the attention, what does this look like operationally for a 1-maintainer project) wasn't really answered.


















