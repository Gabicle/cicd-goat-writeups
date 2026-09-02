# Dodo - Flag

**Captured via:** `wonderland-dodo` build console output, printed in
plaintext (no encoding/decoding step was involved in this challenge,
unlike the others in this set) after the pipeline confirmed the deployed
`dodo` bucket's ACL was genuinely publicly readable.

**Flag:** `A62F...redacted, see 02-attack.md for the retrieval method...`

Redacted here since this is a public repo. As with the previous challenges,
the value itself carries no real confidentiality since it is shared across
every environment running the lab. What's worth documenting in full is the
retrieval method, specifically the two isolated test runs in `02-attack.md`
that pin down exactly which part of the `.checkov.yaml` override actually
mattered.
