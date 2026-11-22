# RFC-0002: Pull request template

RFC PRs here have no fixed shape, so reviewers have to dig for the
problem, the proposal and its impact. Add `.github/pull_request_template.md`
so every PR opens with the same short prompts.

- The template asks for: the RFC number and title, the problem it solves,
  the proposed change, what it affects (specific apps, the whole org,
  or this repo only), and optionally open questions. It has no comment
  period prompt: under RFC-0000 a maintainer starts the period and picks
  its length.
- Authors delete prompts that don't apply. The template guides; it isn't
  enforced.
- This PR adds the template, so merging the RFC adopts it.
- The template is for RFC pull requests in this repository. Pull requests
  in the app repositories follow [pull-requests.md](../standards/pull-requests.md).
- The template is a living document under [RFC-0013](0013-amending-rfcs.md)
  point 5: wording fixes are ordinary pull requests; adding, removing or
  changing a prompt needs an RFC that updates this one.
