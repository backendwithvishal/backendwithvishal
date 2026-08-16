// scripts/update-readme.js
// Updates the <!--START_SECTION--> ... <!--END_SECTION--> blocks in README.md
// with live data: recent GitHub activity + (optionally) latest blog posts from an RSS feed.
//
// Run locally:   GITHUB_USERNAME=Web-dev-vishal node scripts/update-readme.js
// Run in CI:      handled by .github/workflows/update-readme.yml

import fs from 'fs';
import path from 'path';
import Parser from 'rss-parser';

const USERNAME = process.env.GITHUB_USERNAME || 'Web-dev-vishal';
const TOKEN = process.env.GH_TOKEN || process.env.GITHUB_TOKEN; // optional, raises rate limit
const BLOG_RSS_URL = process.env.BLOG_RSS_URL; // optional, e.g. your Medium/Dev.to/Hashnode feed
const README_PATH = path.join(process.cwd(), 'README.md');

const MAX_ACTIVITY = 5;
const MAX_POSTS = 5;

function sectionRegex(name) {
  return new RegExp(`(<!--START_SECTION:${name}-->)([\\s\\S]*?)(<!--END_SECTION:${name}-->)`);
}

async function fetchActivity() {
  const res = await fetch(`https://api.github.com/users/${USERNAME}/events/public`, {
    headers: {
      'User-Agent': USERNAME,
      Accept: 'application/vnd.github+json',
      ...(TOKEN ? { Authorization: `Bearer ${TOKEN}` } : {}),
    },
  });

  if (!res.ok) {
    console.error(`GitHub events fetch failed: ${res.status} ${res.statusText}`);
    return [];
  }

  const events = await res.json();
  const lines = [];

  for (const event of events) {
    if (lines.length >= MAX_ACTIVITY) break;
    const repo = event.repo?.name;
    if (!repo) continue;

    switch (event.type) {
      case 'PushEvent': {
        const count = event.payload?.commits?.length || 1;
        lines.push(`🟢 Pushed ${count} commit${count > 1 ? 's' : ''} to [${repo}](https://github.com/${repo})`);
        break;
      }
      case 'PullRequestEvent': {
        const action = event.payload?.action;
        const num = event.payload?.number;
        const verb = action === 'opened' ? 'Opened' : action === 'closed' ? 'Closed' : action;
        lines.push(`🔀 ${verb} PR #${num} in [${repo}](https://github.com/${repo})`);
        break;
      }
      case 'IssuesEvent': {
        const action = event.payload?.action;
        const num = event.payload?.issue?.number;
        const verb = action === 'opened' ? 'Opened' : 'Closed';
        lines.push(`❗ ${verb} issue #${num} in [${repo}](https://github.com/${repo})`);
        break;
      }
      case 'WatchEvent':
        lines.push(`⭐ Starred [${repo}](https://github.com/${repo})`);
        break;
      case 'CreateEvent':
        if (event.payload?.ref_type === 'repository') {
          lines.push(`🆕 Created repository [${repo}](https://github.com/${repo})`);
        }
        break;
      case 'ForkEvent':
        lines.push(`🍴 Forked [${repo}](https://github.com/${repo})`);
        break;
      default:
        break;
    }
  }

  return lines;
}

async function fetchBlogPosts() {
  if (!BLOG_RSS_URL) return [];
  try {
    const parser = new Parser();
    const feed = await parser.parseURL(BLOG_RSS_URL);
    return (feed.items || []).slice(0, MAX_POSTS).map((item) => `📝 [${item.title}](${item.link})`);
  } catch (err) {
    console.error('Blog RSS fetch failed:', err.message);
    return [];
  }
}

function updateSection(content, name, lines, emptyMessage) {
  const regex = sectionRegex(name);
  if (!regex.test(content)) {
    console.warn(`Marker for section "${name}" not found in README.md — skipping.`);
    return content;
  }
  const body = lines.length ? lines.map((l) => `- ${l}`).join('\n') : emptyMessage;
  return content.replace(regex, `$1\n${body}\n$3`);
}

async function main() {
  if (!fs.existsSync(README_PATH)) {
    console.error('README.md not found at repo root.');
    process.exit(1);
  }

  let content = fs.readFileSync(README_PATH, 'utf8');

  const [activity, posts] = await Promise.all([fetchActivity(), fetchBlogPosts()]);

  content = updateSection(content, 'activity', activity, '_No recent public activity found._');
  content = updateSection(
    content,
    'blog',
    posts,
    '_No blog posts found. Set the `BLOG_RSS_URL` repo secret to enable this section._'
  );

  fs.writeFileSync(README_PATH, content);
  console.log('README.md updated successfully.');
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
