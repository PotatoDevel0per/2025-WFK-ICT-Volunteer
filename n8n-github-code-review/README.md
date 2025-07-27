# Building an Automated AI Code Reviewer with n8n and GitHub 🤖

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Self-Hosting Setup Guide](#self-hosting-setup-guide)
4. [Building the n8n Workflow: A Step-by-Step Guide](#building-the-n8n-workflow)
5. [Testing and Troubleshooting](#testing-and-troubleshooting)
6. [Conclusion and Next Steps](#conclusion-and-next-steps)
7. [Additional Learning Resources](#additional-learning-resources)

---

## Introduction {#introduction}

### What is This Guide?

This guide explains how to use a tool called n8n to create a system that automatically reviews code submitted to GitHub using AI. Think of it as a teacher who automatically grades your homework!

### What We Will Build

We will create a system that performs the following actions:
- When a Pull Request (PR) is sent from a feature branch to the main branch,
- An AI will automatically review the code changes,
- And post review comments directly on the Pull Request.

> **Important!** This system activates when you merge work from a separate branch into `main`, not when you push code directly to the `main` branch.

### Overall Workflow

Here is a high-level overview of the entire automated process. When a developer creates a Pull Request, the system triggers a sequence of actions that result in an AI-generated code review.

1. Modify code in a feature branch
2. Create a Pull Request to main
3. n8n detects the PR via webhook
4. n8n fetches changed file information
5. Data is formatted for the AI
6. AI model reviews the code diffs
7. Review is posted as a comment on the PR
8. 'ReviewedByAI' label is added

---

## Prerequisites {#prerequisites}

1. **GitHub Account** - Where your code is stored (Free).
2. **GitHub Access Token** - A key for n8n to access your GitHub (Free).
3. **A Computer or Server** - To run n8n (self-hosted).
4. **Docker** - A tool to easily install n8n (Free).
5. **Cloudflared** - A tunnel to allow external access to your local server (Free).
6. **Google Gemini API Key** - To act as our AI reviewer (Free tier available).
7. **Claude Pro Account (Optional)** - For a more advanced AI reviewer.

### What is n8n Self-Hosting?

> **Important!** We will be installing and running n8n on our own computer instead of using its cloud service. This is called self-hosting.

For detailed installation instructions, please refer to this guide: 👉 [n8n Self-Hosting Guide](https://github.com/PotatoDevel0per/2025MAS/tree/main/n8n-self-hosting)

**In summary:**
- Install n8n using Docker (Free).
- Set up external access with Cloudflared (Free).
- This allows GitHub webhooks to connect directly to the n8n instance on your computer.

### Getting a GitHub Access Token 🔑

N8N needs a special key (Access Token) to access your GitHub repositories.

**Step-by-step guide:**

1. Click on your GitHub profile (top right corner).
2. Click on **Settings**.
3. Click on **Developer Settings** (bottom of the left menu).
4. Click on **Personal Access Tokens**.
5. Select **Tokens (Classic)**.
6. Click **Generate new token** → **Generate new token (classic)**.
7. Confirm access by entering your password.
8. In the **Note** field, give it a descriptive name (e.g., "N8N Auto Review").
9. Under **Scopes**, check only the `repo` box ✅ (this grants full control of repositories).
10. Click the **Generate token** button.
11. Copy the generated Access Token and save it in a secure place! 📝

> ⚠️ **Important:** The token is shown only once! Copy it and save it to a secure location immediately.

### Estimated Costs

- **n8n (self-hosted):** Free!
- **Docker:** Free!
- **Cloudflared:** Free!
- **Google Gemini API:** Free! (within the free tier limits)
- **Electricity:** A slight increase (if running your computer 24/7).

**Total Cost:** Almost Free! 💰

---

## Self-Hosting Setup Guide {#self-hosting-setup-guide}

> You must complete these steps first!

### Step 1: Create a Cloudflare Tunnel

Run the following command in your terminal:

```bash
cloudflared tunnel --url localhost:5678
```

When you run this, you will get a domain like `https://xxx-xxx-xxx.trycloudflare.com`.

Copy this domain and keep it handy! 🔗

### Step 2: Prepare the Docker Compose File

Create a new folder and a file named `docker-compose.yml` inside it:

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      # Basic settings
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin123
      # Allow external access
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      # Webhook settings (🔗 Paste the domain from Step 1 here!)
      - WEBHOOK_URL=https://your-copied-cloudflare-domain.trycloudflare.com
      # Data storage
      - N8N_USER_FOLDER=/home/node/.n8n
      # Enable AI model integration
      - N8N_AI_ENABLED=true
      # Timezone setting
      - TZ=Asia/Seoul
    volumes:
      - n8n_data:/home/node/.n8n
      - /var/run/docker.sock:/var/run/docker.sock:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  n8n_data:
    driver: local

networks:
  default:
    driver: bridge
```

### Step 3: Run the n8n Container

In the same folder, run:

```bash
docker-compose up -d
```

### Step 4: Verify n8n Access

Navigate to the domain you received in Step 1 (e.g., `https://xxx-xxxxxx.trycloudflare.com`).

Log in with the credentials: `admin` / `admin123`.

> ⚠️ **Important:** The Cloudflare tunnel will close if you close the terminal window. For a permanent solution, you should create a persistent tunnel.

---

## Building the n8n Workflow: A Step-by-Step Guide {#building-the-n8n-workflow}

### Step 1: Setting Up the GitHub Trigger

**What it does:** Connects n8n to GitHub so it can detect when a new Pull Request is created.

**How to set it up:**

1. In n8n, add a "GitHub Trigger" node.
2. Connect your GitHub account (using the Access Token you generated earlier).
3. Select the repository to monitor (e.g., your "test" repository).
4. Set the event type to "pull_request".

**In simple terms:** "Hey n8n, let me know whenever someone requests to merge a branch into main!"

### Step 2: Fetching Changed File Information

**What it does:** Gathers detailed information about which files were changed and how.

**How to set it up:**

1. Add an "HTTP Request" node.
2. Set the URL to: `https://api.github.com/repos/{{$json["repository"]["owner"]["login"]}}/{{$json["repository"]["name"]}}/pulls/{{$json["number"]}}/files`
3. Configure GitHub API authentication (use Header Auth with your token).

**In simple terms:** "Give me the details on exactly which files were changed."

### Step 3: Formatting Data for the AI

**What it does:** Transforms the complex data from GitHub into a simple format that the AI can easily understand.

**Example Code (in a Code node):**

```javascript
// Iterate through each file to build a clean diff string
const files = $input.all().map(item => item.json);
let diffs = '';

for (const file of files) {
  // Add the file name
  diffs += `### File: ${file.filename}\n\n`;
  
  // Add the code changes (patch), handling potential issues
  if (file.patch) {
    // Sanitize the patch to avoid breaking markdown formatting
    const safePatch = file.patch.replace(/```/g, "''");
    diffs += "```diff\n" + safePatch + "\n```\n";
  }
}

// Create the final prompt for the AI
const userMessage = `
You are a senior iOS developer.
Please review the following code changes:

${diffs}

Your task:
- Review the code changes file by file.
- Write comments for the relevant lines of code.
- Ignore files that have no patch.
- Do not repeat the code; write your comments directly.
`;

// Return the formatted message
return { userMessage };
```

**In simple terms:** "Hey AI, here's the code. I've organized it so it's easy for you to read."

### Step 4: Sending the Code to the AI for Review

**What it does:** Sends the formatted code to an AI model to get a professional review.

**How to set it up:**

1. Add an "AI Agent" or "LLM" node.
2. Connect your Google Gemini Chat Model (or another model).
3. Set the system message: "You are an expert code reviewer."
4. Pass the `userMessage` from the previous step as the prompt.

**In simple terms:** "AI teacher, what do you think of this code? Tell me about any problems or improvements."

### Step 5: Posting the Review to GitHub

**What it does:** Takes the AI's review and posts it as a comment on the actual GitHub Pull Request.

**How to set it up:**

1. Add a "GitHub" node and set the operation to "Pull Request" -> "Create a Review".
2. Set the review event type to "COMMENT".
3. Input the AI's response into the `body` field.

**In simple terms:** "AI, post your feedback on the GitHub PR so everyone can see it."

### Step 6: Automatically Adding a Label

**What it does:** Adds a "ReviewedByAI" label to the PR once the review is complete.

**How to set it up:**

1. Add another "GitHub" node and set the operation to "Issue" -> "Edit".
2. In the `Labels` field, add "ReviewedByAI".

**In simple terms:** "Mark this PR to show that the AI has already reviewed it."

---

## Testing and Troubleshooting {#testing-and-troubleshooting}

### How to Test Your Automated Review System

Once everything is set up, you can test the system.

**Test Preparation:**

1. **Create a new branch:**
   ```bash
   git checkout -b feature/test-branch
   ```

2. **Modify some code:**
   Edit any file, for example, add a line to `README.md`.

3. **Commit and push your changes:**
   ```bash
   git add .
   git commit -m "Test: modifying code for AI review"
   git push origin feature/test-branch
   ```

4. **Create a Pull Request on GitHub:**
   Go to your GitHub repository website and create a PR from `feature/test-branch` to `main`.

This will trigger your n8n workflow! 🎉

> **Tip:** The system will not work if you push directly to the `main` branch. You must work in a separate branch and create a PR.

### Troubleshooting Common Issues

#### 1. "The tunnel is not connecting."
- Check if Cloudflared is running in your terminal.
- Verify that your firewall is not blocking port 5678.

#### 2. "I can't access n8n."
- Check if the Docker container is running with `docker ps`.
- Ensure you are accessing the correct local port: `localhost:5678`.

#### 3. "The GitHub webhook isn't working."
- Confirm that the `WEBHOOK_URL` in your `docker-compose.yml` is the correct Cloudflare domain.
- Check the webhook delivery logs in your GitHub repository settings.
- Verify your Access Token has the necessary `repo` permissions.

#### 4. "I'm not getting a response from the AI."
- Check if your Gemini API key is correct.
- Check if you have exceeded your API quota.

#### 5. "GitHub API authentication error."
- Double-check that your Access Token is correct.
- Ensure the token has not expired.

---

## Conclusion and Next Steps {#conclusion-and-next-steps}

### The Benefits of Your New AI Reviewer

- **Cost Savings:** Get automated code reviews for free! 💰
- **Time Savings:** Basic code checks are handled automatically.
- **Consistency:** The AI reviews with the same standards 24/7.
- **Learning Tool:** Improve your coding skills by learning from AI feedback.
- **Efficiency:** Human reviewers can focus on more critical, high-level logic.
- **Skill Development:** Gain DevOps experience by operating your own server.

### Important Precautions

1. **Branching is Mandatory:** Pushing directly to `main` will not trigger the workflow.
2. **Keep Your Server On:** The computer running n8n must be on for it to work.
3. **Security:** Never expose your API keys or tokens publicly.
4. **AI is Not Perfect:** Always have a human review critical code. The AI is an assistant, not a replacement.
5. **Test Small:** Start with small, non-critical changes to test the system.
6. **Electricity Costs:** Running a computer 24/7 may slightly increase your electricity bill.

### Ideas for Further Improvement

- Filter the workflow to only review specific file types (e.g., `.swift`, `.js`).
- Add different labels based on the severity of the issues found.
- Send notifications to Slack or email.
- Compare reviews from multiple AI models.
- Share the same n8n instance with your team members.

---

## Additional Learning Resources {#additional-learning-resources}

- **Detailed n8n Self-Hosting Guide:** [https://github.com/PotatoDevel0per/2025MAS/tree/main/n8n-self-hosting](https://github.com/PotatoDevel0per/2025MAS/tree/main/n8n-self-hosting)
- **Managing Cloudflare Tunnels:** [https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- **n8n Official Documentation:** [https://docs.n8n.io](https://docs.n8n.io)
- **GitHub API Documentation:** [https://docs.github.com/en/rest](https://docs.github.com/en/rest)
- **Docker Basics:** [https://docs.docker.com/get-started/](https://docs.docker.com/get-started/)
- **Docker Compose Guide:** [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

---

🎊 **Congratulations!** You now have the knowledge to build a completely free AI code reviewer!

If self-hosting seems difficult, be sure to follow the detailed guides linked above first.

---

## References

[1] n8n-github-review-guide
https://static-us-img.skywork.ai/prod/analysis/2025-07-27/5363768134482727537/1949339992548777984_4dace926c2f03604aece83f18d76a1b8.md