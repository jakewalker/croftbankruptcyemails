# Conflicts watch list

The `croft-update` skill reads `.croft-conflicts.json` and injects a one-line disclosure under any newsletter `### h3` whose drafted prose or source documents mention any of the match terms below. This file is the human-readable mirror — edit it together with `.croft-conflicts.json` when the conflicts picture changes.

| Rule | Match terms | Disclosure injected |
|------|-------------|---------------------|
| `friends-of-jp` | Friends of JP, FOJP, JP Educational Collective, Jamaica Plain parents | *I represent Friends of JP, a Jamaica Plain parent group that may participate in the case as a lender or claimant.* |
| `croft-bonds` | Croft Bond, Croft Bonds, bondholder, bond holders | *I personally hold Croft Bonds and am represented separately as a bondholder.* |
| `prepaid-tuition` | prepaid tuition, tuition refund, tuition deposit, tuition claim | *I am also a prepaid-tuition claimant and am represented separately on that claim.* |

Matches are case-insensitive and use word boundaries (so "bondholders" matches but "vagabondholder" would not). At most one disclosure is injected per `### h3`. The skill logs how many disclosures it inserted.

A standing disclosure footer pulled from `DISCLOSURE.md` is appended to every issue regardless of these rules.
