blank_issues_enabled: true
contact_links:
  - name: Question
    url: https://github.com/anuraghazra/github-readme-stats/discussions
    about: Please ask and answer questions here.
  - name: Error
    url: https://github.com/anuraghazra/github-readme-stats/issues/1772
    about:
      Before opening a bug report, please check the 'Common Error Codes' issue.
  - name: FAQ
    url: https://github.com/anuraghazra/github-readme-stats/discussions/1770
    about: Please first check the FAQ before asking a question.
import os

file = open('./vercel.json', 'r')
str = file.read()
file = open('./vercel.json', 'w')

str = str.replace('"maxDuration": 10', '"maxDuration": 30')

file.write(str)
file.close()
name: Deployment Prep
on:
  workflow_dispatch:
  push:
    branches:
      - master

jobs:
  config:
    if: github.repository == 'anuraghazra/github-readme-stats'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deployment Prep
        run: python ./.github/workflows/deploy-prep.py
      - uses: stefanzweifel/git-auto-commit-action@v4
        with:
          branch: vercel
          create_branch: true
          push_options: "--force"
name: Test Deployment
on:
  deployment_status:

permissions: read-all

jobs:
  e2eTests:
    if:
      github.repository == 'anuraghazra/github-readme-stats' &&
      github.event_name == 'deployment_status' &&
      github.event.deployment_status.state == 'success'
    name: Perform 2e2 tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm

      - name: Install dependencies
        run: npm ci
        env:
          CI: true

      - name: Run end-to-end tests.
        run: npm run test:e2e
        env:
          VERCEL_PREVIEW_URL: ${{ github.event.deployment_status.target_url }}
import { renderStatsCard } from "../src/cards/stats-card.js";
import { blacklist } from "../src/common/blacklist.js";
import {
  clampValue,
  CONSTANTS,
  parseArray,
  parseBoolean,
  renderError,
} from "../src/common/utils.js";
import { fetchStats } from "../src/fetchers/stats-fetcher.js";
import { isLocaleAvailable } from "../src/translations.js";

export default async (req, res) => {
  const {
    username,
    hide,
    hide_title,
    hide_border,
    card_width,
    hide_rank,
    show_icons,
    include_all_commits,
    line_height,
    title_color,
    ring_color,
    icon_color,
    text_color,
    text_bold,
    bg_color,
    theme,
    cache_seconds,
    exclude_repo,
    custom_title,
    locale,
    disable_animations,
    border_radius,
    number_format,
    border_color,
    rank_icon,
    show,
  } = req.query;
  res.setHeader("Content-Type", "image/svg+xml");

  if (blacklist.includes(username)) {
    return res.send(renderError("Something went wrong"));
  }

  if (locale && !isLocaleAvailable(locale)) {
    return res.send(renderError("Something went wrong", "Language not found"));
  }

  try {
    const stats = await fetchStats(
      username,
      parseBoolean(include_all_commits),
      parseArray(exclude_repo),
    );

    let cacheSeconds = clampValue(
      parseInt(cache_seconds || CONSTANTS.FOUR_HOURS, 10),
      CONSTANTS.FOUR_HOURS,
      CONSTANTS.ONE_DAY,
    );
    cacheSeconds = process.env.CACHE_SECONDS
      ? parseInt(process.env.CACHE_SECONDS, 10) || cacheSeconds
      : cacheSeconds;

    res.setHeader(
      "Cache-Control",
      `max-age=${
        cacheSeconds / 2
      }, s-maxage=${cacheSeconds}, stale-while-revalidate=${CONSTANTS.ONE_DAY}`,
    );

    return res.send(
      renderStatsCard(stats, {
        hide: parseArray(hide),
        show_icons: parseBoolean(show_icons),
        hide_title: parseBoolean(hide_title),
        hide_border: parseBoolean(hide_border),
        card_width: parseInt(card_width, 10),
        hide_rank: parseBoolean(hide_rank),
        include_all_commits: parseBoolean(include_all_commits),
        line_height,
        title_color,
        ring_color,
        icon_color,
        text_color,
        text_bold: parseBoolean(text_bold),
        bg_color,
        theme,
        custom_title,
        border_radius,
        border_color,
        number_format,
        locale: locale ? locale.toLowerCase() : null,
        disable_animations: parseBoolean(disable_animations),
        rank_icon,
        show: parseArray(show),
      }),
    );
  } catch (err) {
    res.setHeader("Cache-Control", `no-cache, no-store, must-revalidate`); // Don't cache error responses.
    return res.send(renderError(err.message, err.secondaryMessage));
  }
};
![Anurag's GitHub stats](https://github-readme-stats.vercel.app/api?username=anuraghazra&show_icons=true)
