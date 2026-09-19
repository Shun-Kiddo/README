name: Update Profile

on:
  schedule:
    - cron: "0 */12 * * *"
  workflow_dispatch:
  push:
    branches: [main]
    paths: [".github/workflows/profile.yml"]

permissions:
  contents: write

jobs:
  stats:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build stats table
        uses: actions/github-script@v7
        with:
          # Optional: add a PAT (read:user scope) as STATS_TOKEN to include private contributions
          github-token: ${{ secrets.STATS_TOKEN || github.token }}
          script: |
            const fs = require('fs');
            const login = context.repo.owner;
            const SPOKEN = 'English, Filipino'; // edit me

            const base = await github.graphql(`
              query($login:String!){
                user(login:$login){
                  createdAt
                  repositories(ownerAffiliations:OWNER, isFork:false, first:100){
                    nodes{ stargazerCount primaryLanguage{ name } }
                  }
                }
              }`, { login });

            const created = new Date(base.user.createdAt);
            const now = new Date();
            const repos = base.user.repositories.nodes;

            const stars = repos.reduce((s, r) => s + r.stargazerCount, 0);
            const langCount = {};
            repos.forEach(r => {
              const n = r.primaryLanguage && r.primaryLanguage.name;
              if (n) langCount[n] = (langCount[n] || 0) + 1;
            });
            const langs = Object.entries(langCount).sort((a, b) => b[1] - a[1]).map(e => e[0]).join(', ') || 'N/A';

            let total = 0, commits = 0, prs = 0, issues = 0;
            const days = new Map();

            for (let y = created.getUTCFullYear(); y <= now.getUTCFullYear(); y++) {
              const from = new Date(Date.UTC(y, 0, 1)).toISOString();
              const to = new Date(Math.min(Date.UTC(y, 11, 31, 23, 59, 59), now.getTime())).toISOString();
              const r = await github.graphql(`
                query($login:String!,$from:DateTime!,$to:DateTime!){
                  user(login:$login){
                    contributionsCollection(from:$from, to:$to){
                      totalCommitContributions
                      totalPullRequestContributions
                      totalIssueContributions
                      contributionCalendar{
                        totalContributions
                        weeks{ contributionDays{ date contributionCount } }
                      }
                    }
                  }
                }`, { login, from, to });
              const c = r.user.contributionsCollection;
              total += c.contributionCalendar.totalContributions;
              commits += c.totalCommitContributions;
              prs += c.totalPullRequestContributions;
              issues += c.totalIssueContributions;
              c.contributionCalendar.weeks.forEach(w =>
                w.contributionDays.forEach(d => days.set(d.date, d.contributionCount)));
            }

            // Streaks
            const sorted = [...days.entries()].sort((a, b) => a[0].localeCompare(b[0]));
            const fmt = s => { const [Y, M, D] = s.split('-'); return `${+M}/${+D}/${Y.slice(2)}`; };

            let longest = 0, longStart = '', longEnd = '', run = 0, runStart = '';
            sorted.forEach(([date, n]) => {
              if (n > 0) {
                if (run === 0) runStart = date;
                run++;
                if (run > longest) { longest = run; longStart = runStart; longEnd = date; }
              } else run = 0;
            });

            let current = 0;
            let i = sorted.length - 1;
            if (i >= 0 && sorted[i][1] === 0) i--; // today may not have activity yet
            while (i >= 0 && sorted[i][1] > 0) { current++; i--; }

            const longestTxt = longest ? `${longest} (${fmt(longStart)} - ${fmt(longEnd)})` : '0';

            const html = `<table align="center">
              <tr>
                <th colspan="2">GitHub Activity</th>
                <th colspan="2">Streaks &amp; Languages</th>
              </tr>
              <tr><th align="left">Total Contributions</th><td>${total}</td><th align="left">Current Streak</th><td>${current}</td></tr>
              <tr><th align="left">Total Commits</th><td>${commits}</td><th align="left">Longest Streak</th><td>${longestTxt}</td></tr>
              <tr><th align="left">Total Pull Requests</th><td>${prs}</td><th align="left">Spoken Languages</th><td>${SPOKEN}</td></tr>
              <tr><th align="left">Total Issues</th><td>${issues}</td><th align="left" rowspan="2">Programming Languages</th><td rowspan="2">${langs}</td></tr>
              <tr><th align="left">Total Stars</th><td>${stars}</td></tr>
            </table>`;

            const readme = fs.readFileSync('README.md', 'utf8');
            const updated = readme.replace(
              /<!--STATS_START-->[\s\S]*?<!--STATS_END-->/,
              `<!--STATS_START-->\n${html}\n<!--STATS_END-->`
            );
            fs.writeFileSync('README.md', updated);

      - name: Commit README
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add README.md
          git diff --cached --quiet || git commit -m "chore: update profile stats"
          git push

  snake:
    runs-on: ubuntu-latest
    steps:
      - uses: Platane/snk/svg-only@v3
        with:
          github_user_name: ${{ github.repository_owner }}
          outputs: |
            dist/github-snake.svg
            dist/github-snake-dark.svg?palette=github-dark

      - uses: crazy-max/ghaction-github-pages@v4
        with:
          target_branch: output
          build_dir: dist
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
