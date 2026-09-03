# Cheshire Cat - Flag

**Captured via:** `wonderland-cheshire-cat` PR build console output, `Read
flag` stage, printed in plaintext (no encoding step was needed, this
challenge reads a file directly rather than binding a masked Jenkins
credential).

**Flag:** `6B31...redacted, see 02-attack.md for the retrieval method...`

Redacted here since this is a public repo. As with the previous challenges,
the value itself carries no real confidentiality since it is shared across
every environment running the lab. What's worth documenting in full is the
retrieval method, specifically identifying the controller node's label via
Jenkins' own REST API and redirecting pipeline execution to it.
