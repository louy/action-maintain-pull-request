# Maintain One Pull Request
A Github Action that maintains a single pull request given a set of head/base branches.

## Use cases
This action is useful for automated changes, especially things that should be constantly updated in a repo.


### Example: Downloading schema from a GraphQL server into a consumer app

Assuming a codebase is using [GraphQL Code Generator](https://the-guild.dev/graphql/codegen/docs/) to generate its consumer code, you can have a pipeline such as:

```
# Regularly keep graphql schema up to date, by running `graphql-codegen` and creating a PR with the changes that auto merges into main
name: Update GraphQL Schema

on:
  schedule:
    - cron: '0 6 * * 1-5' # Workdays 6am
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  update-graphql:
    runs-on: ubuntu-latest
    env:
      EMAIL: github-actions[bot]@users.noreply.github.com
      PR_BRANCH: chore/update-graphql
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@v4
        with:
          # A PAT can be used here if you want Github Actions to trigger on the resulting PR, but it's optional
          token: ${{ secrets.PAT }}

      - uses: actions/setup-node@v4
      - run: npm ci

      - run: npx graphql-codegen

      - id: commit
        name: Commit & Push changes
        run: |
          git checkout -b "${PR_BRANCH}"
          git add .

          if [ -n "$(git status -s)" ]; then
            echo "Adding updated files"
            git commit --no-verify -m "chore: update graphql db" -m "Auto-generated by [${{ github.workflow }} workflow](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})"
            git push --no-verify --force origin "${PR_BRANCH}"
            echo "no-changes=" >> $GITHUB_OUTPUT
          else
            echo "Nothing to commit"
            echo "no-changes=true" >> $GITHUB_OUTPUT
          fi

      - id: pr
        uses: louy/maintain-one-pull-request@v1
        with:
          head_ref: ${{ env.PR_BRANCH }}
          base_ref: main
          pr_title: Update GraphQL Schema
          pr_body: |
            Auto-generated by [${{ github.workflow }} workflow](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
          labels: automated pr,graphql
          delete: ${{ steps.commit.outtputs.no-changes == 'true' }}
          
      - if: ${{ steps.pr.outputs.number != '' }}
        run: gh pr merge ${{ steps.pr.outputs.number }} --auto --squash
```