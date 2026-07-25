# Review PR — moved

The four-agent review (security · Rails patterns · code quality · product domain) is canonical in Beam's plugin, **beam-claude-skills** — a private repo, so this pointer only resolves for Beam engineers.

It has to live there: magicnotes CI checks that repo out and reads `commands/review-pr.md` *as its review instructions*, so the file cannot be a pointer on that side. Keeping a second copy here meant two files that had to stay byte-identical, and they had already started to drift.

If you have Beam access, add the marketplace `https://github.com/wearebeam/beam-claude-skills.git` via `/plugin` and install `beam-claude-skills`. Canonical file: `commands/review-pr.md`.

You probably don't need it on magicnotes anyway — CI reviews every non-draft PR automatically, and running it by hand duplicates that spend.
