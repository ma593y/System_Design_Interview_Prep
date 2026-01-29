# Communication Tips for System Design Interviews

## Overview

Strong communication can make the difference between a good and great system design interview. This guide covers how to effectively present your ideas.

## Think Out Loud

### Why It Matters

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERVIEWER'S PERSPECTIVE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Silent thinking:                                               │
│  "I have no idea what they're considering..."                   │
│  "Are they stuck? Confused? Thinking deeply?"                   │
│  "I can't help or guide them"                                   │
│                                                                  │
│  Thinking out loud:                                             │
│  "Ah, they're weighing trade-offs"                              │
│  "Good, they considered this edge case"                         │
│  "I can redirect them if needed"                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### How to Do It

```
Instead of:                    Say:
────────────────────────────────────────────────────────────────
*silence*                      "Let me think about the data model
                               for a moment..."

*silence*                      "I'm considering two options here:
                               Option A would give us X, but
                               Option B might be better for Y..."

*silence*                      "I need to calculate the storage
                               requirements before deciding..."
```

## Structure Your Explanation

### The STAR Method for Design Decisions

```
S - Situation: What's the context?
T - Task: What are we trying to solve?
A - Action: What's my proposed solution?
R - Result: What are the trade-offs/outcomes?

Example:
"We have high read traffic (Situation) and need sub-100ms latency
(Task). I'll add a Redis cache layer in front of the database
(Action). This reduces DB load and gets us to ~10ms reads, though
we'll need to handle cache invalidation (Result)."
```

### Signal Your Transitions

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRANSITION PHRASES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Starting requirements:                                          │
│  "Let me start by clarifying the requirements..."               │
│  "Before I design, I'd like to understand..."                   │
│                                                                  │
│  Moving to high-level design:                                   │
│  "Now that I understand the requirements, let me sketch         │
│   out the high-level architecture..."                           │
│                                                                  │
│  Deep diving:                                                    │
│  "I'd like to dive deeper into [component]..."                  │
│  "Let's look at how [feature] would work..."                    │
│                                                                  │
│  Discussing trade-offs:                                          │
│  "There's a trade-off here between X and Y..."                  │
│  "We could also approach this with..."                          │
│                                                                  │
│  Wrapping up:                                                    │
│  "To summarize the key points..."                               │
│  "The main bottlenecks to watch are..."                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Explain Trade-offs Clearly

### Framework for Trade-off Discussion

```
"For [decision], I considered [Option A] vs [Option B].

Option A gives us [benefit 1] and [benefit 2],
but comes with [drawback 1].

Option B would provide [different benefit],
however it has [different drawback].

Given our requirement of [key requirement],
I'm choosing Option A because [reasoning].

If requirements change to [alternative requirement],
we might reconsider Option B."
```

### Example

```
"For the database, I considered SQL vs NoSQL.

SQL gives us ACID transactions and complex queries,
but scaling writes horizontally is more challenging.

NoSQL would provide easier horizontal scaling,
however we'd lose strong consistency and JOINs.

Given our requirement of 99.99% availability and
eventual consistency being acceptable for the feed,
I'm choosing Cassandra (NoSQL) for timeline storage.

If we needed financial transactions with strong consistency,
we'd use PostgreSQL instead for those specific tables."
```

## Handle Uncertainty

### When You Don't Know Something

```
DON'T:                         DO:
────────────────────────────────────────────────────────────────
"I don't know"                 "I'm not certain about the exact
                               implementation, but my understanding
                               is... Let me think through the
                               logical approach..."

"I've never done that"         "I haven't implemented this directly,
                               but based on similar problems,
                               I would approach it by..."

Guessing confidently           "I'm making an assumption here that
                               [assumption]. Does that align with
                               what you had in mind?"
```

### When Stuck

```
1. Acknowledge it:
   "I need a moment to think through this..."

2. Break it down:
   "Let me decompose this problem into smaller parts..."

3. Ask for hints:
   "Could you give me a hint about which direction
    you'd like me to explore?"

4. State your thought process:
   "I'm stuck because [reason]. I'm thinking about
    [approach] but not sure about [specific concern]..."
```

## Engage With the Interviewer

### Ask for Feedback

```
Throughout the interview:

"Does this high-level design align with what you expected?"

"I'm about to dive into [component]. Is there a different
 area you'd like me to focus on?"

"I've made several assumptions. Are any of these off base?"

"Is this level of detail appropriate, or should I go deeper?"
```

### Respond to Redirections

```
When interviewer redirects:

Interviewer: "What about handling failures?"

Good response:
"Great point. Let me address failure scenarios.
For [component], we handle failures by..."

Not: "I was going to get to that."
Not: Ignoring and continuing your point
```

### Build on Their Input

```
When interviewer gives a hint:

Interviewer: "Have you considered the case where
              a celebrity tweets?"

Good response:
"Ah yes, the fan-out problem. With millions of followers,
pushing to all timelines would be expensive.
Let me revise the design to handle this case..."
```

## Use Clear Language

### Technical Communication

```
DO:
✓ Use precise terminology
  "We'll use consistent hashing for data partitioning"

✓ Define acronyms first time
  "We'll use a CDN (Content Delivery Network) to cache static assets"

✓ Quantify when possible
  "This reduces latency from 100ms to approximately 10ms"

DON'T:
✗ Use vague language
  "It'll be really fast" → "Target latency is under 50ms"

✗ Overuse jargon without explanation
  "We'll just CQRS it with saga patterns"

✗ Be imprecise about scale
  "A lot of users" → "10 million daily active users"
```

### Confident vs Arrogant

```
Confident:                      Arrogant:
────────────────────────────────────────────────────────────────
"Based on similar systems..."   "Obviously, you would..."

"A common approach is..."       "Everyone knows you should..."

"I'd recommend X because..."    "X is the only right way..."

"One consideration is..."       "The problem with your approach..."
```

## Pace and Time Management

### Managing the 45 Minutes

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIME AWARENESS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Check in with yourself:                                        │
│                                                                  │
│  5 min:  "I should wrap up requirements soon"                   │
│  15 min: "Time to move from high-level to deep dive"           │
│  30 min: "I should start wrapping up the deep dive"            │
│  40 min: "Time to summarize and discuss future work"           │
│                                                                  │
│  If running behind:                                             │
│  "I want to make sure we cover [important topic].              │
│   Let me briefly summarize this part and move on..."           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Pacing Your Speech

```
Too fast:
- Interviewer can't follow
- Seems nervous
- Miss important points

Too slow:
- Run out of time
- Seems unprepared
- Lose engagement

Just right:
- Clear, deliberate pace
- Pause between major points
- Check for understanding
```

## Useful Phrases

### Starting the Interview

```
"Let me start by understanding what we're building..."
"I'll begin with some clarifying questions..."
"Before jumping into the design, I'd like to establish..."
```

### During Design

```
"The key challenge here is..."
"There's an interesting trade-off between..."
"One approach I've seen work well is..."
"Given our constraints, I'd recommend..."
"Let me walk through the data flow..."
```

### Showing Depth

```
"Under the hood, this works by..."
"The reason this matters is..."
"In production, we'd also need to consider..."
"A subtle issue here is..."
```

### Handling Questions

```
"That's a great question. The way I'd handle that is..."
"Let me think about that for a moment..."
"To clarify your question, are you asking about X or Y?"
"That ties into [component] which I'll address by..."
```

### Concluding

```
"To summarize the key design decisions..."
"The main trade-offs we made were..."
"If I had more time, I'd also address..."
"The critical components to monitor would be..."
```

## Body Language (In-Person/Video)

```
DO:
✓ Make eye contact (with camera for video)
✓ Point to diagram while explaining
✓ Nod when understanding feedback
✓ Sit/stand with good posture

DON'T:
✗ Turn away while thinking
✗ Mumble or speak too quietly
✗ Rush through without checking understanding
✗ Show frustration when redirected
```

## Key Takeaways

1. **Think out loud** - Let interviewer follow your reasoning
2. **Structure matters** - Use transitions and frameworks
3. **Trade-offs are key** - Always explain the "why"
4. **Engage actively** - This is a conversation, not a presentation
5. **Stay calm** - It's OK to pause and think
6. **Manage time** - Be aware of pacing

## Practice Exercises

1. Explain a system you've built to a non-technical friend
2. Record yourself doing a mock interview
3. Practice transitioning between topics smoothly
4. Time yourself on different sections

## Further Reading

- "Talking with Tech Leads" by Patrick Kua
- Presentation skills resources
- Mock interview recordings online
