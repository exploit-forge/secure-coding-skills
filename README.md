# secure-coding-skills

An ExploitForge skill for writing and reviewing secure application code and cloud templates. It is aimed at people and coding agents who are adding a feature, reading a pull request, or fixing a finding in Java, C#, Python, TypeScript, PHP, JavaScript, Go, Rust, Ruby, Flutter, or React Native, or in Terraform, Bicep, or CloudFormation.

The checks come from review practice and from bugs we have built and taught: missing owner checks, money stored in one unit and displayed in another, tokens accepted without a real signature, known-bad library pins, and errors that fail open or print a stack to the client. Each rule leads with the pattern to write. A short insecure snippet appears only to show the bug class beside the fix.

This repository does not try to run a multi-agent audit harness. It tells you how to write the safe version and how to report a real defect.

## When to use it

- You are writing a route, query, template, token check, upload, or money movement.
- You are reviewing a pull request in those languages, or a Terraform, Bicep, or CloudFormation change.
- You are fixing a finding and need the smallest change that keeps the legitimate caller working.

## Layout

```
skills/secure-coding-skills/SKILL.md
skills/secure-coding-skills/references/
```

`SKILL.md` is the entry point. The files under `references/` are the rules an agent should open for the category it is touching.

## Install

Copy `skills/secure-coding-skills` into the skills directory your agent loads, or point the agent at this repository and ask it to follow `SKILL.md`.

The skill name is `secure-coding-skills`.

## License

MIT. Copyright (c) 2026 ExploitForge. See [LICENSE](LICENSE).
