# Synthline — Free Email Automation Setup

How to send 150+ cold emails per week without paying a dime.

---

## OPTION 1: GMass (Google Sheets → Gmail) — RECOMMENDED

**Best for:** Quick start, no setup needed
**Cost:** Free tier (50 emails/day)
**What it does:** Sends personalized emails from your Gmail

### Setup (10 minutes)

1. **Install GMass**
   - Go to gmass.co
   - Click "Install GMass for Free"
   - Add to Chrome (requires Gmail account)

2. **Create your lead list in Google Sheets**
   - Go to sheets.new
   - Paste this header row:
     ```
     Name | Email | Company | Industry | City | Notes
     ```
   - Fill in your leads (from lead collector)

3. **Write your email in Gmail**
   - Compose a new email
   - Use `{Name}`, `{Company}`, `{Industry}` as merge tags
   - Example subject: `{Name} — quick question about your calls`

   Example body:
   ```
   Hi {Name},

   Quick question — when someone calls {Company} during
   busy hours or after close, what happens?

   We built an AI phone agent that picks up every call
   and books appointments automatically.

   Free to try. Worth a look?

   Best,
   Alex
   synthline.io
   ```

4. **Set up tracking**
   - BCC: the GMass column in your sheet
   - Enable "Track opens" and "Track clicks"
   - Set "Auto follow-up if no reply: 3 days"

5. **Schedule**
   - Add `Schedule: 9am` to subject line
   - GMass auto-sends at the scheduled time
   - Free limit: 50 emails/day (wait 24h between batches)

### Follow-up Sequence

Create this as a GMass campaign in the same sheet:

| Day | Action |
|-----|--------|
| 1 | Initial email |
| 3 | Follow-up: "Hey {Name} — just following up on my email" |
| 7 | Breakup: "Should I close your file?" |

GMass auto-sends follow-ups only to non-repliers.

---

## OPTION 2: HubSpot Free CRM

**Best for:** Full CRM tracking + deal pipeline
**Cost:** Free forever (up to 1M contacts)

### Setup (20 minutes)

1. **Create account**
   - Go to hubspot.com/products/crm
   - Sign up for free

2. **Import leads**
   - Import tab → Import contacts
   - Upload your CSV (from lead collector)
   - Map columns: Name, Email, Phone, Company

3. **Create deal pipeline**
   - Sales → Deals → Create pipeline
   - Stages: New Lead → Contacted → Demo Booked → Proposal → Closed Won/Lost

4. **Create email templates**
   - Sales → Templates → Create template
   - Use merge tags: `{{ contact.firstname }}`, `{{ contact.company }}`
   - Save 3 templates: Initial, Follow-up, Breakup

5. **Track opens**
   - HubSpot automatically tracks email opens when you send from their platform
   - Free tier includes limited sending (use Gmail integration)

### HubSpot + Gmail Connection
- Install HubSpot Sales Chrome extension
- Sends directly from Gmail but logs everything in HubSpot
- Shows you when prospects open your emails

---

## OPTION 3: Apollo.io

**Best for:** Finding leads + sending in one place
**Cost:** Free tier (10,000 email credits/month, 100 email sends/day)

### Setup (15 minutes)

1. **Create account** at apollo.io
2. **Search leads** (built-in database of 275M contacts)
   - Filter by: Industry, Job Title (owner, manager), Location, Company Size
3. **Create sequence**
   - Sequences → New Sequence
   - Add steps: Email 1 (Day 1), Email 2 (Day 3), Email 3 (Day 7)
4. **Add leads to sequence**
   - Apollo auto-sends on schedule
   - Free tier: 100 sends/day
5. **Track responses**
   - Apollo shows replies, opens, clicks

---

## OPTION 4: Manual (No Tool — Just Gmail)

**Step 1:** Write 30 different emails in a Google Doc
**Step 2:** Open 5 tabs with your lead sheet
**Step 3:** Copy-paste-customize each email
**Step 4:** Send 30 emails in ~45 minutes

**Pro tip:** Create a Gmail filter that labels replies
- Label: "Synthline Replies"
- Never miss a prospect response

### Manual Template (Keep this open while sending)

```
───── COPY FROM HERE ─────
To: [email]
Subject: [Business] — quick question

Hi [Name],

Quick question — when someone calls [Business]
during busy hours or after close, what happens?

We built an AI phone agent that picks up every
call and books appointments automatically.

Free to try. Worth a 10-minute look?

👉 https://synthline.io/#book-demo

Best,
Alex
───── END ─────
```

---

## EMAIL ACCOUNTS

| Provider | Free Limit | Best For |
|----------|-----------|----------|
| Gmail | 500 emails/day | Starting out |
| Outlook | 300 emails/day | Backup account |
| ProtonMail | 150 emails/day | Privacy-focused |
| Yahoo | 500 emails/day | Secondary |

**Strategy:** Create 2-3 free email accounts → rotate them
- alex@... — main (personal brand)
- hello@synthline... — company
- demo@synthline... — demo requests

---

## TRACKING WITHOUT A CRM

Create this in Google Sheets (sheets.new):

```
| Date | Business | Contact | Email Sent | Opened? | Replied? | Called? | Demo? | Closed? | Notes |
```

No tool needed. Just fill it out as you go.

---

## WEEKLY EMAIL SCHEDULE

```
Monday:    30 emails (HVAC/Plumbing) + follow-ups
Tuesday:   30 emails (Dental) + follow-ups
Wednesday: 30 emails (Law) + follow-ups
Thursday:  30 emails (Salons/Auto) + follow-ups
Friday:    30 emails (Medical/Vet) + roll-up
```

With GMass free tier (50/day), break into:
```
9am:    25 emails to Industry A
12pm:   25 emails to Industry B
Next day: follow-ups to non-repliers
```

---

## COMMON ISSUES

**Emails going to spam?**
- Don't send more than 50/day from one Gmail account
- Use your real name, real signature
- Don't use link shorteners
- Warm up a new account by sending 5-10/day for the first week

**No replies?**
- Change subject line (lead with pain, not product)
- Shorten email to 3 sentences max
- Include a specific number/stat
- Call instead of email

**Getting blocked?**
- Take a hint. Move on.
- Add to "Do Not Contact" list
- Don't email from a different account

---

*Synthline — AI Voice Agents for Business*
*Start sending today. Free tools. No excuses.*
