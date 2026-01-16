# 0G Compute Skills for Claude Code

A comprehensive Claude Code skill for working with 0G Compute Network - decentralized AI inference platform.

## What This Skill Does

This skill makes Claude an expert in 0G Compute Network, helping you:

- 🤖 **Build AI inference applications** - Chatbots, image generation, speech-to-text
- 🔧 **Integrate 0G SDK** - Correct API usage with up-to-date patterns
- 💰 **Manage accounts** - Deposits, transfers, refunds with sub-account system
- 📝 **Get production-ready code** - Copy-paste complete example projects

## Installation

### Personal Installation (Recommended)

For use across all your projects:

```bash
# Clone or copy this directory to your personal skills folder
cp -r /path/to/0g-compute-skills ~/.claude/skills/

# Or use git clone
cd ~/.claude/skills
git clone <repository-url> 0g-compute-skills
```

### Project Installation

For sharing with your team:

```bash
# Copy to project skills folder
cp -r /path/to/0g-compute-skills .claude/skills/
```

### Verification

Check if the skill is loaded:

1. Open Claude Code
2. Ask: "What skills are available?"
3. You should see `0g-compute` in the list

## How to Use

### Automatic Activation

The skill activates automatically when you mention relevant keywords:

```
"Help me integrate 0G Compute SDK"
"How do I use broker.inference.processResponse?"
"Create a streaming chatbot with 0G"
```

### Manual Activation

Invoke the skill explicitly:

```
/0g-compute
```

### Example Questions

**For SDK Integration**:
```
"Show me how to initialize the 0G broker"
"What's the correct parameter order for processResponse?"
"Generate a complete chatbot example"
```

**For Specific Services**:
```
"How do I generate images with 0G Compute?"
"Create a speech-to-text transcription app"
"Show me streaming chat implementation"
```

**For Account Management**:
```
"How do I deposit funds to my 0G account?"
"Explain the sub-account system"
"How do I transfer funds to a provider?"
```

## Resources

### External Links

- **0G Compute Documentation**: https://docs.0g.ai
- **SDK GitHub**: https://github.com/0gfoundation/0g-serving-broker
- **Starter Kit**: https://github.com/0gfoundation/0g-compute-ts-starter-kit
- **Discord Community**: https://discord.gg/0glabs
