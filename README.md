# This action is centrally managed in https://github.com/asyncapi/.github/
# Don't make changes to this file in this repo as they will be overwritten with changes made to the same file in above mentioned repo

name: Welcome first time contributors

on:
  pull_request_target:
    types: 
      - opened
  issues:
    types:
      - opened

jobs:
  welcome:
    name: Post welcome message
    if: ${{ !contains(fromJson('["asyncapi-bot", "dependabot[bot]", "dependabot-preview[bot]", "allcontributors[bot]"]'), github.actor) }}
    runs-on: ubuntu-latest
    steps:
    - uses: actions/github-script@v6
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        script: |
          const issueMessage = `Welcome to AsyncAPI. Thanks a lot for reporting your first issue. Please check out our [contributors guide](https://github.com/asyncapi/community/blob/master/CONTRIBUTING.md) and the instructions about a [basic recommended setup](https://github.com/asyncapi/community/blob/master/git-workflow.md) useful for opening a pull request.<br />Keep in mind there are also other channels you can use to interact with AsyncAPI community. For more details check out [this issue](https://github.com/asyncapi/asyncapi/issues/115).`;
          const prMessage = `Welcome to AsyncAPI. Thanks a lot for creating your first pull request. Please check out our [contributors guide](https://github.com/asyncapi/community/blob/master/CONTRIBUTING.md) useful for opening a pull request.<br />Keep in mind there are also other channels you can use to interact with AsyncAPI community. For more details check out [this issue](https://github.com/asyncapi/asyncapi/issues/115).`;
          if (!issueMessage && !prMessage) {
              throw new Error('Action must have at least one of issue-message or pr-message set');
          }
          const isIssue = !!context.payload.issue;
          let isFirstContribution;
          if (isIssue) {
              const query = `query($owner:String!, $name:String!, $contributer:String!) {
              repository(owner:$owner, name:$name){
                issues(first: 1, filterBy: {createdBy:$contributer}){
                  totalCount
                }
              }
            }`;
            const variables = {
              owner: context.repo.owner,
              name: context.repo.repo,
              contributer: context.payload.sender.login
            };
            const { repository: { issues: { totalCount } } } = await github.graphql(query, variables);
            isFirstContribution = totalCount === 1;
          } else {
              const query = `query($qstr: String!) {
                search(query: $qstr, type: ISSUE, first: 1) {
                   issueCount
                 }
              }`;
            const variables = {
              "qstr": `repo:${context.repo.owner}/${context.repo.repo} type:pr author:${context.payload.sender.login}`,
            };
            const { search: { issueCount } } = await github.graphql(query, variables);
            isFirstContribution = issueCount === 1;
          }
          
          if (!isFirstContribution) {
              console.log(`Not the users first contribution.`);
              return;
          }
          const message = isIssue ? issueMessage : prMessage;
          // Add a comment to the appropriate place
          if (isIssue) {
              const issueNumber = context.payload.issue.number;
              console.log(`Adding message: ${message} to issue #${issueNumber}`);
              await github.rest.issues.createComment({
                  owner: context.payload.repository.owner.login,
                  repo: context.payload.repository.name,
                  issue_number: issueNumber,
                  body: message
              });
          }
          else {
            const pullNumber = context.payload.pull_request.number;
              console.log(`Adding message: ${message} to pull request #${pullNumber}`);
              await github.rest.pulls.createReview({
                  owner: context.payload.repository.owner.login,
                  repo: context.payload.repository.name,
                  pull_number: pullNumber,
                  body: message,
                  event: 'COMMENT'
              });
          }
