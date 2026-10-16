---
prompts:
  - prompt: "Our staff should be able to connect their Google Calendar in this Next.js app. Events on the calendars they pick must block their availability and stay up to date without anyone pressing refresh, and every confirmed booking should land in their calendar, with a Google Meet link if they want one. We use Supabase."
    stack: Supabase
  - prompt: "Add Google Calendar sync for our instructors on Firestore. We run group classes: when a student books a class it goes into the instructor's calendar, but the class must stay open for the other students."
    stack: Firestore
  - prompt: "Audit our Google Calendar integration: hosts show as busy at times they cleared weeks ago, some calendars stopped updating without any error, and people who revoked access just see an empty calendar list."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/google-calendar-sync) on timerise.ai.
