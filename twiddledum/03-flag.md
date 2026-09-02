# Twiddledum - Flag

**Captured via:** `wonderland-twiddledum` build console output, base64-encoded
full environment dump printed by the injected line in `twiddledee`'s
`index.js`, decoded and filtered locally.

**Flag:** `7108...redacted, see 02-attack.md for the retrieval method...`

Redacted here since this is a public repo. As with the previous challenges,
the value itself carries no real confidentiality since it is shared across
every environment running the lab. What's worth documenting in full is the
retrieval method, particularly the distinction between publishing a tag via
Gitea's web UI versus pushing one via the git CLI, since that difference is
what actually determined whether the attack worked.
