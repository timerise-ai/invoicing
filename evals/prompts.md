---
prompts:
  - prompt: "Add invoicing to this app: staff issue VAT invoices with numbers that never skip, can issue a correction when one is wrong, see them all in a list, and download each as a PDF."
    stack: No data store
  - prompt: "Our invoices need to go to KSeF. Build the FA(3) file for each invoice we issue and put the KSeF QR code on the PDF."
    stack: Postgres
  - prompt: "Audit the invoices module in this app: can a number be skipped or used twice, and can an invoice change after it was issued?"
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/invoicing) on timerise.ai.
