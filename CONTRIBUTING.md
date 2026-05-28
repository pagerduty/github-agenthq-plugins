# Contributing to PagerDuty AgentHQ Plugins

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Did you find a bug?

- **Do not open up a GitHub issue if the bug is a security vulnerability**, and instead send us an [email](mailto:open-source@pagerduty.com).

- **Ensure the bug was not already reported** by searching on GitHub's [issues](https://github.com/PagerDuty/github-agenthq-plugins/issues) page.

- If you're unable to find an open issue addressing the problem, [open a new one](https://github.com/PagerDuty/github-agenthq-plugins/issues/new?labels=bug&template=bug_report.md). **Use the bug template.**

## Do you intend to add a new feature or change an existing one?

- Check for a similar feature request on the [issues](https://github.com/PagerDuty/github-agenthq-plugins/issues) page.

- If you can't find it, open an issue on GitHub using the [Feature Request](https://github.com/PagerDuty/github-agenthq-plugins/issues/new?labels=enhancement&template=feature_request.md) template.

- Provide as much detail as possible so the request can be analyzed by the team.

- If you aren't sure about the feature you are about to request, reach out to other community members on [PagerDuty Community Forum](https://community.pagerduty.com).

> Note: Features will be reviewed by the core team and discussed with the contributor. Different factors may cause features to be rejected or postponed.

## Pull Requests

Contributions via pull requests are much appreciated but we need you to follow some basic rules so we can work more effectively.

### Step 1: Find something to work on

If you want to contribute a specific feature or fix you have in mind, look at active [pull requests](https://github.com/PagerDuty/github-agenthq-plugins/pulls) to see if someone else is already working on it. If not, please propose that feature request or fix on the [issues page](https://github.com/PagerDuty/github-agenthq-plugins/issues). You will need to reference the issue number on the PR.

On the other hand, if you are here looking for an issue to work on, check out our [backlog of issues](https://github.com/PagerDuty/github-agenthq-plugins/issues) and find something that looks interesting. We label our issues with [GitHub's default labels](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels#about-default-labels). Use that as a reference.

It's a good idea to keep the priority of issues in mind when deciding what to work on. If we have labelled an issue as `low priority`, it means it's something we won't get to work soon while `high priority` issues have a bigger impact, so we are much more likely to give a PR for those issues prompt attention.

### Step 2: Design

We ask you to seek feedback and consensus on your proposed change by iterating on a design document. This is especially useful when you plan a big change or feature, or you want advice on what would be the best path forward.

If you're picking up an existing issue, you can simply post your comment and discuss your proposed changes. If instead you're proposing a new feature, you can start by creating a new [feature request issue](https://github.com/PagerDuty/github-agenthq-plugins/issues/new?labels=enhancement&template=feature_request.md) and discuss your proposed change with the maintainers.

Another way to collect feedback on a new feature request is by sharing it in [PagerDuty's community forum](https://community.pagerduty.com).

### Step 3: Have fun coding

Please make sure you follow these rules:

- Work against the latest source on the **main** branch.
- Try to maintain a single feature or bugfix per pull request. It's okay to introduce a little bit of housekeeping changes along the way, but try to avoid conflating multiple features. Eventually, all these are going to go into a single commit, so you can use that to frame your scope.
- Follow conventional commits guidelines.

### Step 4: Test your changes

Install the plugin locally with Copilot CLI and verify it works as expected:

```bash
# Install from your local clone
copilot plugin install ./path/to/your/local/clone

# Confirm it loaded
copilot plugin list

# Non-interactive smoke check (enumerate loaded agents/skills/MCP servers)
copilot -p "List loaded custom agents, plugin skills, and MCP server names." --allow-all

# Run a skill end-to-end (requires PAGERDUTY_API_KEY in the environment)
copilot -p "Run a pre-commit risk check on this branch against <service-name>"
```

Re-install after every change (`copilot plugin install` is idempotent and reloads the bundle). Uninstall with `copilot plugin uninstall pagerduty` when you're done.

For interactive testing once the Agentic App is wired up, you can also @-mention the app in an issue or assign issues to it.

### Step 5: Pull Request

Once you're done with your changes, you can open a pull request. Make sure to follow the checklist inside the pull request template.

Create a commit with your changes and push them to a new branch or fork, then create a [pull request on GitHub](https://docs.github.com/en/github/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork).

> Note: Core members can push directly to a branch on the repo (following the same conventions detailed below).

Pull request title and message must adhere to [conventional commits](https://www.conventionalcommits.org). This is a summary of the rules:

- The title must begin with `feat: title`, `fix: title`, `refactor: title` or `chore: title`, etc.
- Title should be lowercase.
- No period at the end of the title.

The pull request body should describe _motivation_ and follow the template provided as closely as possible. Think about your code reviewers and what information they need in order to understand what you did.

The body should also include a reference to the issue that this PR is related to in the appropriate section.

Once the pull request is submitted, a reviewer will be assigned by the maintainers.

Discuss review comments and iterate until you get at least one "Approve". When iterating, push new commits to the same branch. Usually, all these are going to be squashed when the maintainers merge to `main`. The commit messages should be hints for you when you finalize your merge commit message.

### Step 6: Merge

Once approved and tested, one of the maintainers will squash-merge to `main` and will use your PR title/description as the commit message. Your name will also be added to the Release Notes of the next release.

## Thank you

PagerDuty AgentHQ Plugins is an open source project and therefore needs the community to support it. We encourage you to help us out!
