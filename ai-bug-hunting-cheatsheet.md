# AI Bug Hunting Cheatsheet - SEC-ACEH

## Target: APK yang pakai AI (ChatGPT, Gemini, Claude)

### 1. Cari Hardcoded API Key (Paling Sering!)
```bash
# di dalam folder jadx
grep -r "sk-" . --include="*.java"
grep -r "openai" . -i
grep -r "api_key" . -i
grep -r "AIza" . # Google Gemini Key
grep -r "xoxb-" . # Slack / AI bot
