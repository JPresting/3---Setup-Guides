# Qwen Code Setup Guide

## Prerequisites
* Node.js 20+ installed
* OpenRouter account with API key

## Installation

```bash
npm install -g @qwen-code/qwen-code
```

## Setup OpenRouter API
1. Get API key from OpenRouter
2. **Option A - Global Windows setup (recommended):**

```cmd
setx OPENAI_API_KEY "your-openrouter-api-key"
setx OPENAI_BASE_URL "https://openrouter.ai/api/v1"
setx OPENAI_MODEL "qwen/qwen3-coder"
```

3. **Option B - Via Qwen console:**

```bash
qwen
/auth
```

Then enter:
* API_KEY: `your-openrouter-api-key`
* BASE_URL: `https://openrouter.ai/api/v1`
* MODEL: `qwen/qwen3-coder`

## Usage
After setting environment variables, **restart your terminal** and run from any folder:

```bash
qwen
```

That's it! The model will work from any directory on your system.

## Verify Setup
Check if variables are set:

```cmd
echo %OPENAI_API_KEY%
echo %OPENAI_BASE_URL%
echo %OPENAI_MODEL%
```

## Troubleshooting
* If credentials aren't working, restart terminal after `setx` commands
* Alternative: Create `.env` file in each project folder with same variables