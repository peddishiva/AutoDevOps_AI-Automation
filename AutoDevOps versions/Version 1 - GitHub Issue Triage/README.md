# AutoDevOps AI Automation --- Version 1

> **Version 1: AI-powered GitHub Issue Triage**\
> An n8n workflow that periodically finds GitHub issues marked for AI
> triage, generates a structured first-pass analysis with an LLM through
> OpenRouter, posts the result as a GitHub issue comment, and removes
> the `ai-triage` label after the comment step.

## Overview

Version 1 is the baseline of the **Universal AutoDevOps AI Automation**
project. It automates an initial issue-triage step so maintainers can
get a consistent starting point for investigating bugs, understanding
missing details, and planning tests.

The workflow was built and manually exercised during development. This
repository copy is a **sanitized workflow export** intended for version
control and reuse; it contains placeholders instead of working
credentials and repository details. Importing it does not connect it to
your GitHub repository automatically.

## What it does

1.  **Runs on a schedule** --- the workflow is configured to check every
    15 minutes when activated.
2.  **Fetches candidate issues** --- requests open issues from the
    configured GitHub repository that have the `ai-triage` label, with a
    page size of up to 10 issues per request.
3.  **Filters pull requests** --- keeps records with an issue number and
    title, and excludes pull requests.
4.  **Generates an AI triage note** --- sends the issue title, number,
    URL, labels, author, and body to an LLM using the OpenRouter
    chat-completions API.
5.  **Posts a comment** --- adds the generated triage note to the GitHub
    issue, with a reminder that it is AI-generated and should be
    verified.
6.  **Removes the triage label** --- attempts to remove `ai-triage`
    after the comment node succeeds.

### AI response structure

The prompt asks the model to return a concise Markdown comment
containing:

-   **Summary**
-   **Likely area**
-   **Reproduction details or clarifying questions**
-   **Suggested investigation steps**
-   **Proposed tests**
-   **Risks and uncertainty**

The prompt also instructs the model not to invent repository-specific
facts or claim that it inspected code it was not given. Since this
version sends issue details rather than repository source code, the
output is a first-pass triage suggestion---not a confirmed diagnosis or
code fix.

## Workflow nodes

  
  ### Node                                     Responsibility
  ---------------------------------------- -----------------------------------
  `Every 15 Minutes`                       Starts a scheduled run every 15
                                           minutes.

  `Fetch Issues Labeled ai-triage`         Calls the GitHub Issues API for
                                           open issues carrying the
                                           `ai-triage` label.

  `Filter Out Pull Requests`               Excludes pull-request records
                                           returned by GitHub's issues
                                           endpoint.

  `AI Triage with OpenAI`                  Calls OpenRouter's chat-completions
                                           endpoint with the configured model
                                           and triage prompt. The node name is
                                           a legacy label; the exported
                                           request uses OpenRouter.

  `Post Triage Comment`                    Posts the generated Markdown
                                           response as a comment on the
                                           matching issue.

  `Remove ai-triage Label After Comment`   Removes the `ai-triage` label after
                                           the preceding comment node
                                           completes successfully.
  ----------------------------------------------------------------------------

## Architecture

``` text
Schedule (every 15 minutes)
          |
          v
Fetch open issues labelled `ai-triage`
          |
          v
Filter out pull requests
          |
          v
OpenRouter LLM: generate triage note
          |
          v
Post comment to GitHub issue
          |
          v
Remove `ai-triage` label
```

## Requirements

-   A running n8n instance with HTTP Request, Code, and Schedule Trigger
    nodes available.
-   A GitHub repository with Issues enabled.
-   A GitHub token with only the permissions required to read issues and
    labels, create issue comments, and remove labels in the target
    repository.
-   An OpenRouter API key and an available chat model.
-   A test repository or test issue for initial validation.

The model configured in this sanitized export is
`nvidia/nemotron-3-ultra-550b-a55b:free`. Model availability, naming,
and free-tier limits can change; select a currently available model in
OpenRouter if needed.

## Setup

### 1. Import the workflow

1.  Open your n8n instance.
2.  Import `workflow.json` from this version folder.
3.  Keep the workflow inactive until configuration and testing are
    complete.

### 2. Configure GitHub settings

In the GitHub HTTP Request nodes, replace the placeholders:

-   `YOUR_GITHUB_OWNER`
-   `YOUR_REPOSITORY`
-   `YOUR_GITHUB_TOKEN`

Use a dedicated, least-privilege token. Do not paste a real token into
the exported JSON or commit it to Git. Prefer n8n credentials or another
supported secret-management mechanism where practical.

The workflow expects issues to be labelled **`ai-triage`**. Add this
label to a test issue to make it eligible for processing.

### 3. Configure OpenRouter

In the AI triage HTTP Request node:

-   Replace `YOUR_OPENROUTER_API_KEY` with a secret managed by n8n where
    possible.
-   Confirm the model identifier is available to your account.
-   Keep the request body and response mapping aligned with the selected
    model/API response format.

### 4. Test before activation

1.  Use a test repository and create a non-production issue.
2.  Add the `ai-triage` label.
3.  Run the workflow manually and inspect each node's input and output.
4.  Confirm the AI response is useful and does not make unsupported
    claims.
5.  Confirm the comment is posted to the correct issue.
6.  Confirm the label is removed only after the comment is successfully
    posted.
7.  Test missing issue descriptions, API failures, rate limits, and
    model errors.
8.  Activate the schedule only after the end-to-end behavior is
    verified.

## Security and operational notes

-   **This export is sanitized.** Replace all placeholders locally;
    never commit actual API keys, access tokens, or private repository
    data.
-   Give the GitHub token the minimum permissions necessary and restrict
    it to the intended repository when possible.
-   Treat issue bodies and other user-submitted issue content as
    untrusted input. The prompt includes defensive instructions, but
    prompt instructions are not a complete security boundary.
-   The workflow posts public or repository-visible comments depending
    on the target repository's visibility and permissions. Review the
    generated output and test in a safe repository first.
-   The workflow does not inspect the repository's source code, create a
    patch, run tests, commit changes, push branches, or open pull
    requests.
-   The `ai-triage` label is removed after the comment node completes;
    monitor execution logs and test failure behavior before production
    use.
-   The sanitized JSON has `active: false`, so it will not run on a
    schedule until you configure and activate it.
-   API/model availability, rate limits, and costs depend on the
    providers and accounts you configure.

## Current scope and limitations

Version 1 focuses on **issue triage only**. It produces an initial
investigation note to help a developer decide what to inspect next.

It does not:

-   read project files or infer the actual root cause from source code;
-   implement a fix or generate a validated patch;
-   execute unit tests or integration tests;
-   create commits, branches, or pull requests;
-   guarantee that AI suggestions are correct.

AI output must be reviewed by a human before being treated as
engineering guidance.

## Versioning

This file documents the Version 1 baseline. Keep each workflow snapshot
in its own folder rather than replacing previous versions.

Suggested layout:

``` text
AutoDevOps versions/
└── Version 1 - GitHub Issue Triage/
    ├── AutoDevOps AI - GitHub Issue Triage.json
    └── README.md
```

When publishing an updated workflow, create a new version folder and
document its changes, configuration, and test status separately. Use Git
history and release tags to track changes over time.

## Troubleshooting

  -----------------------------------------------------------------------
  Problem                             Things to check
  ----------------------------------- -----------------------------------
  No issues are returned              Confirm the owner/repository, that
                                      the issues are open, and that they
                                      carry the exact `ai-triage` label.

  GitHub returns `401` or `403`       Check token validity, repository
                                      access, and token permissions.

  OpenRouter returns an error         Check the API key, model
                                      identifier, account limits, and
                                      request payload.

  The comment is not posted           Inspect the AI node output and
                                      GitHub comment request URL, token
                                      permissions, and issue number
                                      mapping.

  The label is not removed            Check the final node's execution
                                      output and permissions to modify
                                      issue labels.

  Pull requests are processed         Confirm the Code node excludes
  unexpectedly                        records containing `pull_request`.
  -----------------------------------------------------------------------

## License

See the repository-level `LICENSE` file for the project's license terms.
