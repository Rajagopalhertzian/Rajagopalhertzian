name: AI Engineering Profile Tracker

on:
  schedule:
    - cron: "0 */6 * * *"

  workflow_dispatch:

  push:
    paths-ignore:
      - "README.md"

permissions:
  contents: write

jobs:
  update-profile:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Profile
        uses: actions/checkout@v4

      - name: Update Engineering Dashboard
        uses: actions/github-script@v7

        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

          script: |
            const fs = require('fs');

            const username = context.repo.owner;
            const repo = context.repo.repo;

            const repos = await github.paginate(
              github.rest.repos.listForUser,
              {
                username,
                type: 'owner',
                per_page: 100,
                sort: 'updated'
              }
            );

            const ownRepos = repos.filter(
              r => !r.fork && !r.archived
            );

            const languages = new Map();

            const aiKeywords = [
              'agentic', 'agent', 'agents',
              'langchain', 'langgraph',
              'autogen', 'crewai', 'mcp',
              'generative-ai', 'genai',
              'llm', 'rag', 'retrieval',
              'vector-database', 'llama',
              'openai', 'transformers'
            ];

            const techSymbols = {
              Python: '🐍',
              JavaScript: '🟨',
              TypeScript: '🔷',
              Java: '☕',
              C: '⚙️',
              'C++': '⚙️',
              Go: '🔵',
              Rust: '🦀',
              Jupyter: '📓',
              HTML: '🌐',
              CSS: '🎨',
              Shell: '💻'
            };

            const techBadges = {
              pytorch: 'PyTorch',
              tensorflow: 'TensorFlow',
              keras: 'Keras',
              langchain: 'LangChain',
              langgraph: 'LangGraph',
              crewai: 'CrewAI',
              autogen: 'AutoGen',
              mcp: 'MCP',
              yolo: 'YOLO',
              opencv: 'OpenCV',
              fastapi: 'FastAPI',
              django: 'Django',
              docker: 'Docker',
              streamlit: 'Streamlit',
              huggingface: 'Hugging Face',
              transformers: 'Transformers',
              chromadb: 'ChromaDB',
              faiss: 'FAISS',
              llamaindex: 'LlamaIndex'
            };

            const aiProjects = [];

            for (const r of ownRepos) {

              const metadata = [
                r.name,
                r.description || '',
                ...(r.topics || [])
              ].join(' ').toLowerCase();

              const isAI = aiKeywords.some(
                keyword => metadata.includes(keyword)
              );

              if (isAI) {
                aiProjects.push(r);
              }

              const langs = await github.rest.repos
                .listLanguages({
                  owner: username,
                  repo: r.name
                });

              for (const lang of Object.keys(langs.data)) {
                languages.set(
                  lang,
                  (languages.get(lang) || 0) + 1
                );
              }
            }

            const sortedLanguages = [...languages.entries()]
              .sort((a, b) => b[1] - a[1]);

            const techRows = sortedLanguages.map(
              ([name, count]) => {
                const symbol = techSymbols[name] || '⚡';

                return `| **${name}** | ${symbol} | ${count} |`;
              }
            );

            const detectedTopics = new Set();

            for (const r of ownRepos) {
              for (const topic of r.topics || []) {
                const key = topic.toLowerCase();

                if (techBadges[key]) {
                  detectedTopics.add(techBadges[key]);
                }
              }
            }

            const topicBadges = [...detectedTopics]
              .sort()
              .map(name => {
                const slug = name
                  .toLowerCase()
                  .replaceAll(' ', '-');

                return `![${name}](https://img.shields.io/badge/${encodeURIComponent(name)}-technology-334155?style=flat-square)`;
              });

            const technologySection = [
              ...techRows,
              '',
              topicBadges.join(' ')
            ].join('\n');

            const aiRows = aiProjects
              .sort((a, b) =>
                new Date(b.pushed_at) - new Date(a.pushed_at)
              )
              .slice(0, 10)
              .map(r => {
                const date = r.pushed_at
                  ? new Date(r.pushed_at)
                      .toISOString().slice(0, 10)
                  : 'N/A';

                const topics = (r.topics || [])
                  .slice(0, 4)
                  .join(', ');

                const stack = topics || r.language || 'See repository';

                return `| [${r.name}](https://github.com/${username}/${r.name}) | ${stack} | ${date} |`;
              });

            const techContent = technologySection ||
              '| No repositories detected | ⚙️ | 0 |';

            const aiContent = aiRows.length
              ? aiRows.join('\n')
              : '| No matching AI repositories yet | ⚙️ | -- |';

            const readmePath = 'README.md';

            let readme = fs.readFileSync(
              readmePath,
              'utf8'
            );

            function replaceSection(start, end, content) {

              const pattern = new RegExp(
                `<!-- ${start}:START -->[\\s\\S]*?<!-- ${start}:END -->`
              );

              if (!pattern.test(readme)) {
                throw new Error(
                  `Missing README markers: ${start}`
                );
              }

              readme = readme.replace(
                pattern,
                `<!-- ${start}:START -->\n${content}\n<!-- ${start}:END -->`
              );
            }

            replaceSection(
              'TECH-STACK',
              'TECH-STACK',
              techContent
            );

            replaceSection(
              'AI-PROJECTS',
              'AI-PROJECTS',
              aiContent
            );

            fs.writeFileSync(readmePath, readme);

            console.log(
              `Repositories: ${ownRepos.length}`
            );

            console.log(
              `AI projects: ${aiProjects.length}`
            );

            console.log(
              `Languages detected: ${languages.size}`
            );

      - name: Commit Dashboard Updates
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git add README.md

          git diff --cached --quiet || (
            git commit -m "Update live AI engineering dashboard"
            git push
          )
