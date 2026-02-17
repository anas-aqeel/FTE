# Gemini Prompt Templates

**Last Updated:** 2026-02-18

This document contains all prompt templates used for Gemini API interactions throughout the system.

---

## Table of Contents

1. [Email Classification](#email-classification)
2. [Event Classification](#event-classification)
3. [Assignment Classification](#assignment-classification)
4. [Classroom Announcement Classification](#classroom-announcement-classification)
5. [WhatsApp Summarization](#whatsapp-summarization)
6. [Conversational Agent](#conversational-agent)

---

## Email Classification

**Used by:** Gmail Ingestion Worker
**Purpose:** Classify, summarize, and score importance of emails

### Prompt Template:

```python
EMAIL_CLASSIFICATION_PROMPT = """
You are an AI assistant that classifies emails for a student's personal assistant system.

Analyze this email and extract the following information:

Email Details:
- Subject: {subject}
- From: {sender}
- Received: {received_at}
- Body: {body}

Tasks:
1. Generate a concise summary (1-2 sentences)
2. Classify into ONE category: task, announcement, meeting, personal, spam
3. Assign importance score (0.0 to 1.0):
   - 1.0: Urgent, deadline-related, from professor/authority
   - 0.7-0.9: Important but not urgent
   - 0.4-0.6: Moderate importance
   - 0.0-0.3: Low importance, informational
4. Extract any mentioned deadlines (date/time if present)

Return ONLY valid JSON (no markdown, no explanation):
{{
  "summary": "Brief summary here",
  "category": "announcement",
  "importance_score": 0.8,
  "extracted_deadline": "2024-03-15T23:59:00Z",
  "reasoning": "Why this score was assigned"
}}

IMPORTANT:
- Be conservative with high scores - only assign 0.9+ for truly urgent items
- Return null for extracted_deadline if no deadline mentioned
- Category must be one of: task, announcement, meeting, personal, spam
"""
```

### Example Usage:

```python
def classify_email(email: Dict) -> Dict:
    prompt = EMAIL_CLASSIFICATION_PROMPT.format(
        subject=email['subject'],
        sender=email['sender'],
        received_at=email['received_at'],
        body=email['body'][:2000]  # Truncate long emails
    )

    response = gemini_model.generate_content(prompt)
    return json.loads(response.text)
```

---

## Event Classification

**Used by:** Gmail/Calendar Event Extraction
**Purpose:** Classify calendar events and assign importance

### Prompt Template:

```python
EVENT_CLASSIFICATION_PROMPT = """
Classify this calendar event for a student:

Event Details:
- Title: {title}
- Start: {start_time}
- End: {end_time}
- Description: {description}
- Location: {location}

Assign importance score based on:
- Required attendance: higher score
- Academic events (exams, classes): higher score
- Social events: moderate score
- Optional events: lower score

Classify event type (choose ONE):
- class: Regular class sessions
- meeting: Team meetings, office hours
- deadline: Submission deadlines, exam times
- exam: Tests, quizzes, exams
- social: Club events, social gatherings
- personal: Personal appointments

Return ONLY valid JSON:
{{
  "importance_score": 0.75,
  "event_type": "class",
  "reasoning": "Required class session"
}}
"""
```

---

## Assignment Classification

**Used by:** Google Classroom Ingestion Worker
**Purpose:** Classify assignments and assess importance

### Prompt Template:

```python
ASSIGNMENT_CLASSIFICATION_PROMPT = """
You are classifying a Google Classroom assignment for a student.

Assignment Details:
- Course: {course_name}
- Title: {title}
- Description: {description}
- Due Date: {due_date}
- Materials: {materials}

Tasks:
1. Assign importance score (0.0 to 1.0):
   - 1.0: Major exams, final projects, critical deadlines (due within 48 hours)
   - 0.7-0.9: Regular assignments, quizzes, important homework
   - 0.4-0.6: Practice problems, optional readings
   - 0.0-0.3: Supplementary materials, reference documents

2. Classify assignment type (choose ONE):
   - exam: Tests, quizzes, midterms, finals
   - project: Long-term projects, presentations
   - homework: Regular assignments, problem sets
   - reading: Reading assignments, textbook chapters
   - practice: Optional practice problems
   - other: Miscellaneous

3. Estimate effort level (choose ONE):
   - high: 5+ hours expected
   - medium: 2-5 hours expected
   - low: < 2 hours expected

Consider factors:
- Due date proximity (sooner = higher score)
- Assignment type (exam > project > homework > reading)
- Point value if mentioned
- Required vs optional
- Complexity indicators in description

Return ONLY valid JSON:
{{
  "importance_score": 0.85,
  "assignment_type": "project",
  "estimated_effort": "high",
  "reasoning": "Major project worth 30% of grade, due in 3 days"
}}
"""
```

---

## Classroom Announcement Classification

**Used by:** Google Classroom Ingestion Worker
**Purpose:** Classify announcements from instructors

### Prompt Template:

```python
CLASSROOM_ANNOUNCEMENT_PROMPT = """
Classify this Google Classroom announcement:

Announcement Details:
- Course: {course_name}
- Title: {title}
- Text: {text}
- Posted By: {announced_by}
- Posted At: {announced_at}

Assign importance based on:
- Contains deadlines or date changes: high importance (0.8-1.0)
- Exam/quiz announcements: high importance (0.8-1.0)
- Project updates or clarifications: medium-high (0.6-0.8)
- General reminders: moderate importance (0.4-0.6)
- Supplementary info: low importance (0.0-0.4)

Classify category (choose ONE):
- urgent: Time-sensitive, requires immediate action
- deadline: Mentions specific due dates or deadlines
- exam: Exam-related announcements
- change: Schedule or requirement changes
- general: General updates and reminders
- informational: Supplementary information

Extract any key information mentioned:
- Date/time changes
- New deadlines
- Important requirements

Return ONLY valid JSON:
{{
  "importance_score": 0.9,
  "category": "deadline",
  "extracted_info": "Midterm exam rescheduled to March 25th at 2 PM",
  "reasoning": "Critical schedule change affecting all students"
}}
"""
```

---

## WhatsApp Summarization

**Used by:** WhatsApp Service Hourly Summarization
**Purpose:** Summarize 1-hour batches of WhatsApp messages

### Prompt Template:

```python
WHATSAPP_SUMMARY_PROMPT = """
You are summarizing a 1-hour batch of WhatsApp messages for a student.

Chat: {chat_name}
Type: {"Group chat" if is_group else "Private conversation"}
Time Window: {batch_start_time} to {batch_end_time}
Message Count: {message_count}

Messages:
{messages}

Tasks:
1. Generate a brief summary (2-3 sentences) of the main discussion
2. Extract key points (list 3-5 most important items, empty list if casual chat)
3. Identify any mentioned deadlines, assignments, or action items
4. Assign importance score:
   - 1.0: Contains urgent deadlines or critical information that requires immediate action
   - 0.7-0.9: Important project updates, decisions, or time-sensitive information
   - 0.4-0.6: Regular updates, discussions, or moderate importance
   - 0.0-0.3: Casual conversation, no action items or important information

Guidelines:
- If the chat is just casual conversation, set importance_score low (0.0-0.3)
- Only extract deadlines if explicitly mentioned with dates
- Key points should be actionable or informative
- Empty key_points array is acceptable for casual chats

Return ONLY valid JSON:
{{
  "summary": "Discussion about final project deadline and meeting location",
  "key_points": [
    "Final project due March 20th",
    "Team meeting moved to Room 301",
    "Need to submit draft by Friday"
  ],
  "mentioned_deadlines": [
    {{"task": "Final project submission", "deadline": "2024-03-20T23:59:00Z"}},
    {{"task": "Draft submission", "deadline": "2024-03-15T23:59:00Z"}}
  ],
  "importance_score": 0.9,
  "reasoning": "Contains critical deadline information and meeting logistics"
}}

If no important information:
{{
  "summary": "Casual conversation about weekend plans",
  "key_points": [],
  "mentioned_deadlines": [],
  "importance_score": 0.2,
  "reasoning": "No actionable items or important information"
}}
"""
```

### Message Formatting:

```python
def format_messages_for_prompt(messages: List[Dict]) -> str:
    """Format messages for inclusion in prompt"""
    formatted = []
    for msg in messages:
        timestamp = datetime.fromisoformat(msg['timestamp']).strftime('%H:%M')
        sender = msg['sender']
        text = msg['body'][:200]  # Truncate long messages
        formatted.append(f"[{timestamp}] {sender}: {text}")
    return "\n".join(formatted)
```

---

## Conversational Agent

**Used by:** Conversational Agent with Gemini Integration
**Purpose:** Answer user queries using retrieved data

### System Prompt:

```python
CONVERSATION_SYSTEM_PROMPT = """
You are a personal assistant AI helping a student stay organized and on top of their academic commitments.

Your name: Assistant (but call the user "Brother")
Your role: Help the user manage schedules, deadlines, assignments, and communications

Personality traits:
- Friendly and supportive (use "Brother" as a term of endearment)
- Concise but thorough (avoid unnecessary verbosity)
- Proactive (mention upcoming deadlines without being asked)
- Empathetic (acknowledge academic stress)
- Reliable (admit when you don't know something)

Available information:
You have access to the user's:
1. Gmail emails (including professor communications)
2. Google Calendar events (classes, meetings)
3. Google Classroom assignments and announcements
4. WhatsApp message summaries (group chats and private messages)

When answering queries:
1. Synthesize information from multiple sources
2. Prioritize by urgency and importance
3. Mention source when relevant ("According to your email from Professor Smith...")
4. If information is missing or unclear, say so explicitly
5. Suggest proactive actions ("Would you like me to remind you tomorrow?")
6. Format responses clearly (use bullet points for multiple items)
7. Always provide context (dates, times, sources)

Example responses:
- "Brother, you have 3 things due this week: Math homework (Monday), Essay draft (Wednesday), and Project presentation (Friday)"
- "Based on your WhatsApp group chat, the team meeting location changed to Room 301 at 2 PM"
- "I don't see any information about that exam in your records. Should I check your recent emails again?"
- "Brother, you have an urgent deadline today! Your assignment for Database Systems is due at 11:59 PM"

Current date and time: {current_datetime}
User's timezone: {user_timezone}
"""
```

### Query Prompt Template:

```python
CONVERSATION_QUERY_PROMPT = """
{system_prompt}

---
CONVERSATION HISTORY:
{conversation_history}

---
AVAILABLE DATA (retrieved from user's records):

{retrieved_data}

---
USER QUERY: {user_query}

---
INSTRUCTIONS:
Provide a helpful, natural language response based on the available data above.
- Use conversation history for context
- Reference specific data points when relevant (mention sources)
- Be concise but complete
- If the query cannot be answered with available data, say so clearly and suggest alternatives
- Format lists with bullet points for readability
- Always include relevant dates and times

RESPONSE:
"""
```

### Data Formatting Functions:

```python
def format_conversation_history(messages: List[Dict]) -> str:
    """Format conversation history for prompt"""
    if not messages:
        return "(No previous messages in this conversation)"

    history = []
    for msg in messages[-10:]:  # Last 10 messages
        role = "User" if msg['role'] == 'user' else "Assistant"
        history.append(f"{role}: {msg['content']}")

    return "\n".join(history)


def format_retrieved_data(data: Dict) -> str:
    """Format retrieved data for context in prompt"""
    sections = []

    # Emails
    if data.get('emails'):
        emails_text = "EMAILS:\n"
        for email in data['emails'][:5]:
            emails_text += f"""
- From: {email['sender']}
  Subject: {email['subject']}
  Received: {format_datetime(email['received_at'])}
  Summary: {email['summary']}
  Importance: {email['importance_score']:.2f}/1.0
"""
        sections.append(emails_text)

    # Events
    if data.get('events'):
        events_text = "CALENDAR EVENTS:\n"
        for event in data['events'][:10]:
            events_text += f"""
- {event['title']}
  Time: {format_datetime(event['start_time'])} to {format_datetime(event['end_time'])}
  Location: {event.get('location', 'Not specified')}
  Type: {event['event_type']}
  Importance: {event['importance_score']:.2f}/1.0
"""
        sections.append(events_text)

    # Assignments
    if data.get('assignments'):
        assignments_text = "ASSIGNMENTS:\n"
        for assignment in data['assignments'][:10]:
            assignments_text += f"""
- {assignment['course_name']}: {assignment['title']}
  Due: {format_datetime(assignment['due_date'])}
  Status: {assignment['status']}
  Importance: {assignment['importance_score']:.2f}/1.0
  Description: {assignment['description'][:150]}...
"""
        sections.append(assignments_text)

    # Announcements
    if data.get('announcements'):
        announcements_text = "IMPORTANT ANNOUNCEMENTS:\n"
        for announcement in data['announcements'][:5]:
            announcements_text += f"""
- {announcement['title']}
  From: {announcement['announced_by']}
  Date: {format_datetime(announcement['announced_at'])}
  Category: {announcement['category']}
  Importance: {announcement['importance_score']:.2f}/1.0
  Content: {announcement['content'][:150]}...
"""
        sections.append(announcements_text)

    # WhatsApp Summaries
    if data.get('whatsapp_summaries'):
        whatsapp_text = "WHATSAPP MESSAGE SUMMARIES:\n"
        for summary in data['whatsapp_summaries'][:5]:
            chat_type = "Group" if summary['is_group'] else "Private"
            whatsapp_text += f"""
- Chat: {summary['chat_name']} ({chat_type})
  Time: {format_datetime(summary['batch_start_time'])} to {format_datetime(summary['batch_end_time'])}
  Summary: {summary['summary']}
  Key Points: {', '.join(summary.get('key_points', [])[:3])}
  Importance: {summary['importance_score']:.2f}/1.0
"""
        sections.append(whatsapp_text)

    if not sections:
        return "(No relevant data found in user's records for this query)"

    return "\n\n".join(sections)


def format_datetime(dt_string: str) -> str:
    """Format datetime for human-readable display"""
    dt = datetime.fromisoformat(dt_string.replace('Z', '+00:00'))
    return dt.strftime('%A, %B %d at %I:%M %p')
```

### Complete Prompt Construction:

```python
def construct_conversation_prompt(
    user_query: str,
    conversation_history: List[Dict],
    retrieved_data: Dict,
    current_datetime: str,
    user_timezone: str = "UTC"
) -> str:
    """Construct full prompt for Gemini API"""

    system_prompt = CONVERSATION_SYSTEM_PROMPT.format(
        current_datetime=current_datetime,
        user_timezone=user_timezone
    )

    history_text = format_conversation_history(conversation_history)
    data_context = format_retrieved_data(retrieved_data)

    return CONVERSATION_QUERY_PROMPT.format(
        system_prompt=system_prompt,
        conversation_history=history_text,
        retrieved_data=data_context,
        user_query=user_query
    )
```

---

## Error Handling

### Gemini API Error Responses:

```python
# When Gemini API fails
GEMINI_ERROR_RESPONSE = {
    "summary": "Error processing content",
    "importance_score": 0.5,  # Default to medium
    "reasoning": "AI processing failed, manual review recommended"
}

# When content is blocked
CONTENT_BLOCKED_RESPONSE = {
    "summary": "Content filtered by safety settings",
    "importance_score": 0.3,
    "reasoning": "Content was blocked by safety filters"
}

# When JSON parsing fails
JSON_PARSE_ERROR = {
    "summary": "Failed to parse AI response",
    "importance_score": 0.5,
    "reasoning": "AI returned invalid format"
}
```

---

## Best Practices

1. **Always request JSON format** - Specify "Return ONLY valid JSON" to avoid markdown formatting
2. **Include examples** - Show expected output format in prompts
3. **Be specific about scores** - Give clear scoring guidelines (0.0-1.0 scale)
4. **Limit context size** - Truncate long emails/messages to stay within token limits
5. **Handle failures gracefully** - Return default values if AI classification fails
6. **Log AI responses** - Store raw responses for debugging and improvement
7. **Version prompts** - Track prompt versions in code comments
8. **Test edge cases** - Verify prompts handle empty data, special characters, etc.

---

## Token Management

### Estimated Token Usage:

- **Email Classification**: ~500-1000 tokens per email
- **Event Classification**: ~300-500 tokens per event
- **Assignment Classification**: ~400-800 tokens per assignment
- **WhatsApp Summarization**: ~800-1500 tokens per hour batch
- **Conversational Query**: ~1000-3000 tokens per query (varies by data retrieved)

### Cost Optimization:

1. Truncate long content before sending to API
2. Batch classification when possible
3. Cache common classifications
4. Use lower temperature (0.3) for consistent results
5. Implement rate limiting to avoid quota exhaustion

---

## Testing Prompts

### Test Cases:

```python
# Test email classification
test_email = {
    "subject": "URGENT: Assignment Deadline Extended",
    "sender": "professor@university.edu",
    "body": "The deadline for Assignment 3 has been extended to March 20th.",
    "received_at": "2024-03-15T10:00:00Z"
}

# Expected output: importance_score > 0.8, category = "announcement"

# Test casual WhatsApp
test_messages = [
    {"sender": "Alice", "body": "Hey what's up?", "timestamp": "..."},
    {"sender": "Bob", "body": "Not much, you?", "timestamp": "..."}
]

# Expected output: importance_score < 0.3, empty key_points
```

---

**Note:** All prompts should be reviewed and refined based on real-world performance. Monitor AI responses and adjust scoring thresholds as needed.
