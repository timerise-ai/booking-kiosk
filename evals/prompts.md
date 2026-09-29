---
prompts:
  - prompt: "Build a self-service booking kiosk for our karting track: pick a session, a time slot, the number of karts and the driver names, then pay at the counter. Go back to the start after two minutes without a touch."
    stack: Firestore
  - prompt: "Add kiosk mode to our booking app: a touchscreen flow with an on-screen keyboard, pay by QR code, and a find-my-booking screen to change a reservation."
  - prompt: Audit our kiosk against the skill's hard rules and list every place it breaks one.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/booking-kiosk) on timerise.ai.
