# Study Group Guide

A complete guide for practicing system design with peers. Includes mock interview formats, feedback frameworks, and study group structures.

---

## Why Practice with Others

Solo practice has limits. Practicing with peers provides:

- **Realistic pressure** - Simulates interview environment
- **Blind spots revealed** - Others catch what you miss
- **Different perspectives** - Learn alternative approaches
- **Communication practice** - Explaining is a skill
- **Accountability** - Regular schedule keeps you on track
- **Motivation** - Shared journey is easier

---

## Study Group Formation

### Ideal Group Size
- **Minimum:** 2 people (interview pairs)
- **Optimal:** 3-4 people (rotate roles, diverse perspectives)
- **Maximum:** 6 people (beyond this, split into subgroups)

### Finding Study Partners
- Colleagues preparing for interviews
- Online communities (Discord, Reddit, LinkedIn)
- Coding bootcamp alumni
- University alumni networks
- Mock interview platforms

### Group Composition Tips
- Similar experience levels work best
- Mix of backgrounds provides diverse perspectives
- Committed availability is more important than skill level
- Remote-friendly tools make geography irrelevant

---

## Meeting Structure

### Recommended Schedule
- **Frequency:** 2-3 times per week
- **Duration:** 90-120 minutes per session
- **Consistency:** Same days/times each week

### Session Format (2 hours)

| Time | Activity | Description |
|------|----------|-------------|
| 0:00-0:05 | Check-in | Brief updates, energy level |
| 0:05-0:50 | Mock Interview 1 | First person as candidate |
| 0:50-1:00 | Feedback 1 | Structured feedback for first candidate |
| 1:00-1:05 | Break | Quick reset |
| 1:05-1:50 | Mock Interview 2 | Second person as candidate |
| 1:50-2:00 | Feedback 2 + Wrap-up | Feedback and next steps |

For groups of 3-4:
- Rotate: Interviewer, Candidate, Observer(s)
- Observer provides additional feedback
- Extend session or do multiple sessions per week

---

## Mock Interview Format

### Roles

**Interviewer Responsibilities:**
- Present the problem clearly
- Ask clarifying questions if candidate doesn't
- Provide hints if stuck (sparingly)
- Redirect if going off-track
- Push back on weak points
- Manage time

**Candidate Responsibilities:**
- Drive the conversation
- Ask clarifying questions
- State assumptions
- Think out loud
- Draw diagrams
- Discuss tradeoffs

**Observer Responsibilities (if applicable):**
- Take notes on both interviewer and candidate
- Track time usage
- Note communication patterns
- Prepare specific feedback

### Interview Flow (45 minutes)

```
[0:00-0:05] Problem Introduction
- Interviewer presents the problem
- Keep it vague initially (like real interviews)

[0:05-0:10] Requirements Clarification
- Candidate asks questions
- Interviewer answers or says "you decide"

[0:10-0:15] Capacity Estimation
- Candidate estimates scale
- Interviewer validates or corrects

[0:15-0:25] High-Level Design
- Candidate draws architecture
- Interviewer asks clarifying questions

[0:25-0:40] Deep Dive
- Focus on 1-2 components
- Interviewer guides to interesting areas

[0:40-0:45] Wrap-up
- Candidate summarizes
- Interviewer asks final questions
```

---

## Problem Bank for Sessions

### Beginner Problems (Weeks 1-2)
1. URL Shortener
2. Pastebin
3. Rate Limiter
4. Unique ID Generator
5. Key-Value Store

### Intermediate Problems (Weeks 3-4)
1. Twitter Feed
2. Instagram
3. Web Crawler
4. Notification System
5. Chat System

### Advanced Problems (Weeks 5+)
1. YouTube
2. Google Docs
3. Uber
4. Distributed Message Queue
5. Search Engine

### Problem Selection Strategy
- Match problem difficulty to candidate's level
- Rotate who picks the problem
- Occasionally do surprise problems
- Revisit problems after improvement

---

## Feedback Framework

### The SBI Model (Situation-Behavior-Impact)

Structure feedback as:
- **Situation:** When you were [doing X]...
- **Behavior:** You [specific action]...
- **Impact:** Which made me [feel/think/understand]...

**Example:**
"When you were explaining the database choice (situation), you listed three options and immediately chose one without explaining tradeoffs (behavior). This made it seem like you hadn't fully considered alternatives (impact)."

### Feedback Categories

#### 1. Requirements & Scoping
- Did they ask clarifying questions?
- Were assumptions reasonable?
- Was scope appropriate for time?

#### 2. Technical Design
- Was the architecture sound?
- Were component choices justified?
- Were tradeoffs discussed?

#### 3. Communication
- Did they think out loud?
- Were explanations clear?
- Did diagrams help understanding?

#### 4. Problem-Solving
- How did they handle ambiguity?
- Did they recover from mistakes?
- How did they respond to pushback?

#### 5. Time Management
- Did they allocate time well?
- Did they go too deep too early?
- Did they cover enough breadth?

---

## Feedback Templates

### Quick Feedback Form (5 minutes)

```
Candidate: _______________
Problem: _______________
Date: _______________

STRENGTHS (What went well):
1. _______________
2. _______________

IMPROVEMENTS (What to work on):
1. _______________
2. _______________

OVERALL IMPRESSION:
[ ] Not Ready  [ ] Getting There  [ ] Ready  [ ] Strong

ONE THING TO FOCUS ON NEXT:
_______________
```

### Detailed Feedback Form (10 minutes)

```
Candidate: _______________
Problem: _______________
Date: _______________
Interviewer: _______________

REQUIREMENTS PHASE (Score 1-5): ___
- Asked clarifying questions: Y / N / Partially
- Made reasonable assumptions: Y / N / Partially
- Scoped appropriately: Y / N / Partially
Notes: _______________

CAPACITY ESTIMATION (Score 1-5): ___
- Attempted estimation: Y / N
- Numbers were reasonable: Y / N / Partially
- Showed work: Y / N
Notes: _______________

HIGH-LEVEL DESIGN (Score 1-5): ___
- Clear diagram: Y / N / Partially
- Covered major components: Y / N / Partially
- Justified choices: Y / N / Partially
Notes: _______________

DEEP DIVE (Score 1-5): ___
- Showed depth of knowledge: Y / N / Partially
- Discussed tradeoffs: Y / N / Partially
- Handled follow-ups well: Y / N / Partially
Notes: _______________

COMMUNICATION (Score 1-5): ___
- Thought out loud: Y / N / Partially
- Structured explanation: Y / N / Partially
- Responded well to feedback: Y / N / Partially
Notes: _______________

TIME MANAGEMENT (Score 1-5): ___
- Completed all phases: Y / N
- Appropriate depth at each phase: Y / N / Partially
Notes: _______________

TOTAL SCORE: ___ / 30

TOP STRENGTHS:
1. _______________
2. _______________
3. _______________

PRIORITY IMPROVEMENTS:
1. _______________
2. _______________
3. _______________

WOULD HIRE FOR SYSTEM DESIGN ROLE:
[ ] Strong No  [ ] No  [ ] Lean No  [ ] Lean Yes  [ ] Yes  [ ] Strong Yes
```

### Interviewer Self-Assessment

```
How did I do as interviewer?

PROBLEM PRESENTATION:
- Was problem clear? Y / N
- Did I give appropriate info? Y / N

GUIDANCE:
- Did I help without giving away? Y / N
- Did I redirect effectively? Y / N

DIFFICULTY CALIBRATION:
- Was difficulty appropriate? Y / N
- Did I adjust based on candidate? Y / N

WHAT I'LL DO DIFFERENTLY NEXT TIME:
_______________
```

---

## Calibration Sessions

### Purpose
Ensure all group members have similar expectations for "good" designs.

### Calibration Exercise

1. **Watch together:** Find a system design video on YouTube
2. **Score independently:** Each person scores using the feedback form
3. **Compare scores:** Discuss differences
4. **Align on standards:** Agree on what each score means

### Sample Calibration Discussion Questions
- What makes a design "good enough" vs "excellent"?
- How much depth is expected at each level?
- When should hints be given?
- What's the bar for a passing interview?

---

## Interviewer Guide

### How to Present Problems

**Good:**
"Design a system like Twitter where users can post short messages and see posts from people they follow. Let's start with you asking any clarifying questions."

**Bad:**
"Design Twitter. It needs to handle 500 million users, support real-time updates, and have a recommendation engine. Here's the schema I'm thinking..."

### Hint Levels

**Level 1 - Nudge:**
"Have you thought about what happens when..."
"What are some options for storing this data?"

**Level 2 - Direction:**
"Consider thinking about the read vs write ratio."
"Usually in this scenario, we'd think about caching."

**Level 3 - Direct Help:**
"Let me suggest looking at fan-out strategies."
"One common approach here is to use..."

**Rule of thumb:** Start with Level 1. Only escalate if stuck for 2+ minutes.

### Challenge Questions Bank

Use these to probe deeper:

**Scale challenges:**
- "What if traffic increases 10x?"
- "How does this work with 1 billion users?"

**Failure scenarios:**
- "What if this service goes down?"
- "How do you handle network partitions?"

**Tradeoff probes:**
- "Why did you choose X over Y?"
- "What are the downsides of this approach?"

**Edge cases:**
- "What about users in Australia? (latency)"
- "What if someone has 50 million followers?"

---

## Progress Tracking

### Individual Tracking

Each member maintains a log:

```
Week: ___
Problems Practiced: _______________
Strongest Performance: _______________
Biggest Learning: _______________
Focus for Next Week: _______________
```

### Group Tracking

Track group progress:

| Member | Sessions | Avg Score | Trend | Ready? |
|--------|----------|-----------|-------|--------|
| | | | | |
| | | | | |
| | | | | |

---

## Common Pitfalls & Solutions

### Pitfall 1: Being Too Nice
**Problem:** Feedback is always positive, no one improves
**Solution:** Agree that honest feedback is a gift. Use the framework to ensure specificity.

### Pitfall 2: Inconsistent Attendance
**Problem:** People skip sessions, momentum lost
**Solution:** Set expectations upfront. Minimum attendance requirement. Pair with accountability buddy.

### Pitfall 3: Same Problems Repeatedly
**Problem:** Boredom, false confidence
**Solution:** Rotate who picks problems. Use the problem bank. Occasionally do "surprise" problems.

### Pitfall 4: Feedback Without Action
**Problem:** Same mistakes repeated
**Solution:** Start each session with review of last feedback. Track improvement on specific items.

### Pitfall 5: Unbalanced Participation
**Problem:** Same person always interviewer or always candidate
**Solution:** Strict rotation. Track role assignments.

---

## Remote Practice Setup

### Tools

**Video Conferencing:**
- Zoom, Google Meet, or Discord
- Camera on for engagement
- Good audio is essential

**Collaborative Whiteboarding:**
- Excalidraw (free, simple)
- Miro or FigJam (feature-rich)
- tldraw (free, real-time)

**Note-taking:**
- Shared Google Doc for feedback
- Notion for tracking progress

### Remote Best Practices
- Test tools before sessions
- Have backup communication (phone numbers)
- Use screen share for diagrams
- Take turns sharing screen
- Record sessions (with consent) for self-review

---

## 8-Week Study Group Curriculum

### Week 1-2: Foundations
**Goals:** Establish process, practice basics
**Problems:** URL Shortener, Rate Limiter
**Focus:** Requirements gathering, basic architecture

### Week 3-4: Core Patterns
**Goals:** Master common patterns
**Problems:** Twitter Feed, Key-Value Store
**Focus:** Caching, database design, fan-out

### Week 5-6: Complex Systems
**Goals:** Handle multi-component systems
**Problems:** Chat System, File Storage
**Focus:** Real-time, sync, consistency

### Week 7-8: Advanced & Polish
**Goals:** Ready for any problem
**Problems:** YouTube, Uber, or novel problems
**Focus:** Communication, handling unknowns

### Weekly Structure
- **Session 1:** New problem introduction + mock
- **Session 2:** Same problem, switch roles
- **Session 3:** Review learnings, calibration

---

## Sample Meeting Agenda

```
SYSTEM DESIGN STUDY GROUP
Date: _______________
Attendees: _______________

1. CHECK-IN (5 min)
   - How's everyone feeling?
   - Any wins from last session?

2. REVIEW LAST SESSION (5 min)
   - What feedback did we act on?
   - Progress updates

3. MOCK INTERVIEW 1 (50 min)
   - Candidate: _______________
   - Interviewer: _______________
   - Problem: _______________

4. FEEDBACK 1 (10 min)
   - Use feedback template
   - Specific, actionable items

5. BREAK (5 min)

6. MOCK INTERVIEW 2 (50 min)
   - Candidate: _______________
   - Interviewer: _______________
   - Problem: _______________

7. FEEDBACK 2 (10 min)

8. WRAP-UP (5 min)
   - Key learnings today
   - Action items for next session
   - Confirm next meeting

TOTAL: 2 hours
```

---

## Accountability Agreements

Have each member commit to:

```
STUDY GROUP COMMITMENT

I, _______________, commit to:

1. Attending at least ___% of scheduled sessions
2. Giving notice at least 24 hours if I can't attend
3. Completing assigned preparation before sessions
4. Providing honest, constructive feedback
5. Being open to receiving feedback
6. Tracking my progress weekly
7. Supporting other members' growth

Signed: _______________
Date: _______________
```

---

## Graduation Criteria

Group members are "interview ready" when they can:

- [ ] Complete any beginner problem confidently
- [ ] Complete most intermediate problems competently
- [ ] Attempt advanced problems with reasonable progress
- [ ] Receive "Lean Yes" or better on feedback forms consistently
- [ ] Self-assess their performance accurately
- [ ] Give useful feedback to others

Celebrate when members achieve these milestones and get actual offers!
