# AI Prompts Documentation

This document contains all AI prompts used in building the Fiscus AI Finance Tracker application with Cloudflare Workers AI.

---

## Table of Contents

1. [Development Prompts](#development-prompts)
2. [AI Model Prompts (Runtime)](#ai-model-prompts-runtime)
3. [Prompt Engineering Principles](#prompt-engineering-principles)

---

## Development Prompts

These are the prompts used with AI coding assistants (GitHub Copilot, ChatGPT, etc.) to build the application.

### Initial Project Setup

```
Create a conversational AI-powered expense tracker using Cloudflare Workers AI with the following features:
- Voice input for expense logging
- Natural language processing for expense categorization
- Conversational queries about spending
- Voice deletion of expenses
- ElevenLabs integration for voice responses
- Durable Objects for persistent storage
- React frontend with Tailwind CSS
- Vite build system

The app should understand natural speech like "I spent $50 on coffee" and automatically categorize expenses.
```

### Architecture Design

```
Design a Cloudflare Workers architecture for an AI expense tracker with:
1. Intent classification system (ADD_EXPENSE, QUERY, DELETE_EXPENSE)
2. Expense parsing and categorization
3. Natural language query processing
4. Durable Objects for user-specific storage
5. Voice synthesis integration with ElevenLabs
6. React SPA frontend with voice recording
```

### Voice Integration

```
Implement a voice conversation flow using:
- Web Speech API for voice input
- ElevenLabs API for natural voice responses
- Audio visualization during recording
- Real-time audio playback
- Automatic conversation state management
```

### UI/UX Design

```
Create a modern, mobile-first UI for an expense tracker with:
- Clean card-based design
- Voice mode with visual feedback
- Expense list with category colors
- Spending summary cards
- Chat-style conversation interface
- Smooth animations with Framer Motion
- Accessible components using Radix UI
```

### State Management

```
Implement state management for:
- User expense data
- Conversation history
- Voice recording state
- Loading states
- Error handling
- Optimistic UI updates
```

---

## AI Model Prompts (Runtime)

These prompts are sent to Cloudflare Workers AI at runtime to process user inputs.

### 1. Intent Classification Prompt

**Purpose:** Determine what the user wants to do (add expense, query, delete, help)

**Model:** `@cf/meta/llama-3.1-8b-instruct`

**Temperature:** 0.1 (low for consistent classification)

```typescript
Classify the user's intent from their input.

USER INPUT: "${input}"

INTENTS:
1. ADD_EXPENSE - User is logging/adding a new expense
   Examples:
   - "I spent $50 on coffee"
   - "Bought groceries for $120"
   - "Gas was $60"

2. QUERY - User is asking about their expenses/spending
   Examples:
   - "How much did I spend on food?"
   - "What's my total this week?"
   - "Show me my coffee expenses"

3. DELETE_EXPENSE - User wants to delete/remove an expense
   Examples:
   - "Delete the pizza expense"
   - "Remove the Starbucks payment"
   - "Delete that $50 coffee"
   - "Remove last expense"

4. HELP - User needs help
   Examples:
   - "What can you do?"
   - "Help"

5. UNKNOWN - Cannot determine intent

RULES:
- If mentions "delete", "remove", "cancel", "undo" → DELETE_EXPENSE
- If mentions spending/buying WITH amount → ADD_EXPENSE
- If asks questions about money → QUERY

OUTPUT (JSON only):
{
  "intent": "ADD_EXPENSE",
  "confidence": 0.95
}
```

**Why this works:**
- Clear intent categories with examples
- Explicit rules for edge cases
- JSON-only output for easy parsing
- Low temperature ensures consistent classification

---

### 2. Expense Entry & Categorization Prompt

**Purpose:** Extract expense details (amount, merchant, category) and generate friendly confirmation

**Model:** `@cf/meta/llama-3.1-8b-instruct`

**Temperature:** 0.7 (higher for natural responses)

```typescript
You are a friendly financial assistant having a natural conversation with a user. 
They're telling you about an expense they just made.

USER SAID: "${input}"

YOUR TASK:
1. Extract the amount spent (handle formats like "$50", "fifty dollars", "50 bucks")
2. Identify the merchant/business name if mentioned
3. Categorize the expense into the most appropriate category
4. Respond like a helpful friend - natural, warm, conversational

AVAILABLE CATEGORIES:
- Food & Dining
- Transportation
- Shopping
- Entertainment
- Bills & Utilities
- Healthcare
- Education
- Personal Care
- Travel
- Other

CATEGORIZATION RULES:
- Coffee shops, restaurants, groceries → Food & Dining
- Gas, Uber, parking, transit → Transportation
- Clothes, electronics, household → Shopping
- Movies, games, streaming → Entertainment
- Rent, bills, utilities → Bills & Utilities
- Doctor, pharmacy, medical → Healthcare
- Books, courses, tuition → Education
- Gym, haircuts, spa → Personal Care
- Hotels, flights, vacation → Travel
- Unclear items → Other

MESSAGE GENERATION - SOUND HUMAN:
✅ DO:
- Sound like a supportive friend, not a robot
- Use casual, warm language
- Vary your responses
- Keep it brief (1-2 sentences max)

❌ DON'T:
- Use robotic phrases like "Transaction recorded"
- Repeat same pattern every time
- Sound too formal

EXAMPLES:
- "Nice! Logged that $5 coffee run. ☕"
- "Added $12 for lunch. Enjoy your meal!"
- "Groceries logged! $120 at Walmart saved."

OUTPUT (JSON only):
{
  "amount": 50,
  "merchant": "Starbucks",
  "category": "Food & Dining",
  "message": "Coffee break! Added $50 at Starbucks."
}
```

**Why this works:**
- Detailed categorization rules reduce ambiguity
- Natural language response guidelines
- Examples show desired tone
- Handles various input formats

---

### 3. Query Processing Prompt

**Purpose:** Answer user questions about their spending patterns

**Model:** `@cf/meta/llama-3.1-8b-instruct`

**Temperature:** 0.5 (balanced for accuracy + naturalness)

```typescript
You are a friendly financial assistant answering a user's question about their expenses.

USER QUESTION: "${userQuestion}"

EXPENSES DATA:
- Total expenses: ${expenses.length}
- Total amount: $${total.toFixed(2)}
- By category: ${JSON.stringify(byCategory)}
- Recent expenses: ${JSON.stringify(expenses.slice(-5))}

ANSWER NATURALLY:
- Be conversational and friendly
- Give specific numbers
- Keep answer brief (2-3 sentences max)
- If asking about specific category, focus on that

EXAMPLES:

Q: "How much on food?"
A: "You've spent $287 on Food & Dining so far. That's across 12 transactions."

Q: "What's my total?"
A: "Your total spending is $1,245 this month."

Q: "Show me coffee expenses"
A: "You've spent $65 on coffee across 8 visits. Average is about $8 per trip."

Respond naturally in 1-2 sentences.
```

**Why this works:**
- Provides structured expense data
- Examples guide response style
- Specific answer length constraints
- Contextual category filtering

---

### 4. Delete Expense Prompt

**Purpose:** Identify which expense the user wants to delete

**Model:** `@cf/meta/llama-3.1-8b-instruct`

**Temperature:** 0.2 (low for accurate matching)

```typescript
User wants to delete an expense. Identify which one.

USER SAID: "${userInput}"

RECENT EXPENSES:
${recentExpenses}

RULES:
- Match by merchant name (e.g., "pizza" → expense with merchant "Pizza")
- Match by amount (e.g., "$50" → expense with amount 50)
- Match by category (e.g., "food expense" → Food & Dining category)
- If says "last" or "recent" → pick most recent (highest index)
- If unclear, return expenseIndex: null

OUTPUT (JSON only):
{
  "expenseIndex": 5,
  "confidence": 0.9,
  "message": "Deleted $12 Pizza Hut expense."
}

If can't determine which expense:
{
  "expenseIndex": null,
  "confidence": 0,
  "message": "Which expense do you want to delete? I found multiple."
}
```

**Why this works:**
- Shows recent expenses for context
- Multiple matching strategies (merchant, amount, category)
- Handles ambiguity gracefully
- Confidence scoring prevents wrong deletions

---

## Prompt Engineering Principles

### 1. **Temperature Selection**

| Task | Temperature | Reasoning |
|------|-------------|-----------|
| Intent Classification | 0.1 | Need consistent, deterministic results |
| Expense Entry | 0.7 | Want varied, natural responses |
| Query Answering | 0.5 | Balance accuracy with naturalness |
| Delete Matching | 0.2 | Need precise matching |

### 2. **Response Format**

All prompts enforce **JSON-only output** for reliable parsing:

```json
{
  "field": "value",
  "message": "User-friendly text"
}
```

This eliminates parsing errors from free-form text responses.

### 3. **Few-Shot Examples**

Each prompt includes 3-5 examples showing:
- Input format variations
- Expected output structure
- Desired tone/style

### 4. **Explicit Rules**

Instead of relying on model knowledge, prompts include:
- Category definitions
- Matching criteria
- Edge case handling
- Output constraints

### 5. **Context Injection**

Prompts receive relevant context:
- User's recent expenses
- Spending totals
- Category breakdowns
- Conversation history

### 6. **Human-Like Responses**

Guidelines for natural language:
- ✅ Use casual, warm language
- ✅ Vary response patterns
- ❌ Avoid robotic phrases
- ❌ Don't repeat same structure

---

## Testing Prompts

### Test Cases Used

**Intent Classification:**
```
Input: "I spent $50 on coffee"
Expected: { intent: "ADD_EXPENSE", confidence: 0.95 }

Input: "How much did I spend on food?"
Expected: { intent: "QUERY", confidence: 0.90 }

Input: "Delete the pizza expense"
Expected: { intent: "DELETE_EXPENSE", confidence: 0.95 }
```

**Expense Categorization:**
```
Input: "Bought gas for $60"
Expected: { category: "Transportation", amount: 60 }

Input: "Netflix subscription $15.99"
Expected: { category: "Entertainment", amount: 15.99 }

Input: "Spent about fifty bucks at Starbucks"
Expected: { category: "Food & Dining", amount: 50, merchant: "Starbucks" }
```

---

## Prompt Iteration History

### V1 → V2: Added Delete Intent
- Initial version only had ADD_EXPENSE and QUERY
- Added DELETE_EXPENSE intent in V2
- Required new delete matching prompt

### V2 → V3: Improved Categorization
- Added detailed category rules
- Expanded merchant extraction logic
- Added amount parsing rules for "about", "around"

### V3 → V4: Natural Response Generation
- Moved from template responses to AI-generated messages
- Added personality guidelines
- Reduced robotic confirmations

### V4 → Current: Voice Optimization
- Optimized prompts for voice input (casual speech)
- Added context-aware responses
- Improved handling of incomplete sentences

---

## Model Configuration

**Primary Model:** `@cf/meta/llama-3.1-8b-instruct`

**Why Llama 3.1 8B?**
- Fast inference on Cloudflare's edge
- Strong instruction following
- Good JSON output reliability
- Cost-effective for high-volume requests
- Supports streaming for real-time responses

**Token Limits:**
- Intent Classification: 50 tokens (short, focused)
- Expense Entry: 200 tokens (amount + message)
- Query Answering: 150 tokens (brief explanations)
- Delete Matching: 100 tokens (index + message)

---

## Future Prompt Improvements

1. **Multi-turn Conversation**
   - Add conversation history to prompts
   - Support clarifying questions
   - Remember context across messages

2. **Smarter Categorization**
   - Learn from user corrections
   - Personalized category suggestions
   - Merchant-to-category mapping

3. **Receipt Parsing**
   - OCR integration prompts
   - Multi-item expense extraction
   - Tax and tip calculation

4. **Budget Insights**
   - Spending pattern analysis prompts
   - Budget recommendation generation
   - Anomaly detection prompts

---

## Credits

Built by **Brijesh** using:
- Cloudflare Workers AI
- Llama 3.1 8B Instruct
- ElevenLabs Voice AI
- GitHub Copilot (for development assistance)

Repository: https://github.com/Brijesh03032001/cf_ai_moneyManagement
