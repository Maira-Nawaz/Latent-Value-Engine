# Latent Value Engine — Automated B2B Lead Generation & Outreach Pipeline

> Built with n8n · Serper.dev · Groq (Llama 3.3) · Google Sheets

![Pipeline Diagram](https://raw.githubusercontent.com/Maira-Nawaz/Latent-Value-Engine/main/pipeline-diagram.jpeg)

---

## What is This Project?

The **Latent Value Engine** is a fully automated B2B sales pipeline that discovers dental clinics with no chatbot capabilities and generates personalised WhatsApp outreach messages for each one — ready to send in a single click.

It was originally designed to target dental clinics in Islamabad, Pakistan, but the logic is fully adaptable to any niche or city.

---

## The Core Idea

Many dental clinics have websites with static appointment forms. Patients visit at night, can't get answers, and leave without booking. That lost revenue is **latent value** — hidden money the clinic doesn't even know it's losing.

This pipeline:
1. Finds those clinics automatically
2. Scans their websites to confirm they have no chatbot
3. Extracts their contact details
4. Writes a personalised sales pitch proving you've done your homework
5. Saves everything to a CRM (Google Sheets) with a one-click WhatsApp send button

---

## Pipeline Architecture

```
Trigger
   ↓
The Scout          → Searches Google Maps for 20 dental clinics via Serper.dev
   ↓
Data Unroller      → Breaks bulk response into individual clinic records
   ↓
Qualified Lead Filter → Keeps only clinics with 50+ reviews and a website
   ↓
Website Deep Scan  → Fetches raw HTML from each clinic website
   ↓
Chatbot Detector   → Filters out clinics already using chatbot tools
   ↓
HTML Extractor     → Pulls clinic name, phone, form URL, WhatsApp link
   ↓
Loop Over Items    → Processes one clinic at a time (batch size: 1)
   ↓
AI Agent (Groq)    → Writes personalised 60-word sales pitch per clinic
   ↓
Add to CRM         → Saves all data + pitch to Google Sheets
   ↓
Wait (5s)          → Pauses to respect API rate limits
   ↓
Back to Loop       → Repeats for every clinic
```

---

## Tech Stack

| Tool | Role | Cost |
|---|---|---|
| [n8n](https://n8n.io) | Workflow automation engine | Free tier available |
| [Serper.dev](https://serper.dev) | Google Maps search API | 2,500 free searches |
| [Groq Console](https://console.groq.com) | AI inference (Llama 3.3-70b) | Free tier available |
| [Google Sheets](https://docs.google.com/spreadsheets/d/1NHmzMRwTw3Pl1VSjhh64LdmrySNznO8X1NxygOaVdVE/edit?gid=0#gid=0) | CRM / lead storage | Free |

---

## Node Configuration

### Node 1: Manual Trigger
Starts the workflow when you click Execute.

---

### Node 2: The Scout (HTTP Request)
```
Method:  POST
URL:     https://google.serper.dev/maps
Headers:
  X-API-KEY:    <your_serper_api_key>
  Content-Type: application/json
Body:
  {
    "q": "Dental Clinics in Islamabad",
    "limit": 20
  }
```

---

### Node 3: Data Unroller (Code — JavaScript)
```javascript
return $input.all()[0].json.places.map(place => ({ json: place }));
```

---

### Node 4: Qualified Lead Filter
```
Condition 1: {{ $json.ratingCount }} greater than 50
AND
Condition 2: {{ $json.website }} is not empty
```

---

### Node 5: Website Deep Scan (HTTP Request)
```
Method:          GET
URL:             {{ $json.website }}
Response Format: Text
Output Field:    data
Ignore SSL:      ON
Never Error:     ON
```

---

### Node 6: Chatbot Detector (Filter)
```
{{ $json.data }} does not contain → intercom
{{ $json.data }} does not contain → drift.com
{{ $json.data }} does not contain → zendesk
{{ $json.data }} does not contain → tidio
{{ $json.data }} does not contain → shifa
{{ $json.data }} does not contain → pims.gop

All conditions joined with AND
```

---

### Node 7: HTML Extractor
```
Operation:     Extract HTML Content
Source Data:   JSON
JSON Property: data

Extraction Values:
  Clinic_Name → CSS: title               → Return: Text
  Phone       → CSS: a[href*="tel:"]     → Return: Text
  Form_type   → CSS: form                → Return: Attribute (action)
  Whatsapp    → CSS: a[href*="whatsapp"] → Return: Attribute (href)
```

---

### Node 8: Loop Over Items
```
Batch Size: 1
```

---

### Node 9: AI Agent (Groq — Llama 3.3-70b-versatile)

**User Message Prompt:**
```
Write a short, punchy outreach message for {{ $json.Clinic_Name }}.

CONTEXT: I noticed they have a manual appointment form at {{ $json.Form_type }} 
and a contact number of {{ $json.Phone }}.

RULES:
- No formal greetings like "Dear Sir/Madam". Use "Hi [Clinic Name] team," or just "Hi,"
- Mention their booking form is static and doesn't answer patient questions after hours
- Briefly mention a "Digital Front Desk" that handles WhatsApp questions and books appointments
- End with "Is this something you've looked into before?" or "Open to a quick 2-minute demo?"
- Do not use the same phrasing for every lead
- Sometimes call it "AI Receptionist", other times "Smart Booking Assistant" or "24/7 Dental Concierge"
- Max 60 words. No emojis.
```

**System Message:**
```
You are a local business growth specialist in Islamabad. Your tone is professional, 
helpful, and direct. You hate corporate jargon. You speak like a person who just 
visited their website and noticed a small leak in their business that you know how 
to fix. Use British English spelling.
```

---

### Node 10: Add to CRM (Google Sheets)

CRM Sheet: [Dental Leads CRM](https://docs.google.com/spreadsheets/d/1NHmzMRwTw3Pl1VSjhh64LdmrySNznO8X1NxygOaVdVE/edit?gid=0#gid=0)

```
Operation:            Append or Update Row
Mapping Column Mode:  Map Each Column Manually

Column Mappings:
  Clinic Name        → {{ $('Loop Over Items').item.json.Clinic_Name }}
  Website URL        → {{ $('Qualified Lead Filter').item.json.website }}
  Phone Number       → {{ $('Loop Over Items').item.json.Phone }}
  Form Action        → {{ $('HTML').item.json.Form_type }}
  WhatsApp Link      → {{ $('HTML').item.json.Whatsapp }}
  Sales Pitch        → {{ $json.output }}
  WhatsApp Send Link → https://wa.me/{{ $('Loop Over Items').item.json.Phone }}?text={{ encodeURIComponent($json.output) }}
```

**Google Sheet Column Headers (Row 1):**
```
Clinic Name | Website URL | Phone Number | Form Action | WhatsApp Link | Sales Pitch | WhatsApp Send Link
```

---

### Node 11: Wait
```
Resume:      After Time Interval
Wait Amount: 5
Wait Unit:   Seconds
```
Connect the output of this node back to **Loop Over Items** to complete the loop.

---

## Setup Guide

### Step 1: Prerequisites
Create accounts and get API keys for:
- [n8n.io](https://n8n.io) — free account
- [Serper.dev](https://serper.dev) — copy your API key
- [Groq Console](https://console.groq.com) — create and copy an API key
- Google account for Google Sheets

### Step 2: Google Sheet Setup
Create a new Google Sheet named **Dental Leads CRM** with these exact column headers in Row 1:

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| Clinic Name | Website URL | Phone Number | Form Action | WhatsApp Link | Sales Pitch | WhatsApp Send Link |

### Step 3: Build the Workflow in n8n
Follow the node configurations above in order. Connect each node left to right, and connect the Wait node's output back to Loop Over Items.

### Step 4: Add Credentials
- **Serper:** Add your API key as a header in The Scout node
- **Groq:** Create a Groq credential in the AI Agent node using your API key
- **Google Sheets:** Authenticate with your Google account in the Add to CRM node

### Step 5: Quick Import
Rather than building from scratch, you can import the workflow directly:
1. Download `workflow.json` from this repo
2. Open n8n
3. Click the three dots menu top right
4. Click **Import**
5. Select the downloaded `workflow.json`
6. Add your own API keys for Serper, Groq and Google Sheets
7. Click Execute Workflow

### Step 6: Run It
Click **Execute Workflow** and watch your Google Sheet populate automatically with qualified leads and personalised pitches.

---

## Expected Output

After running the workflow you will have a Google Sheet with rows like this:

| Clinic Name | Website URL | Phone | Form Action | WhatsApp Link | Sales Pitch | WhatsApp Send Link |
|---|---|---|---|---|---|---|
| Oradent Dental Clinic | oradentdentalclinic.com | +923249134745 | /appointment-submit | api.whatsapp.com/... | "Hi Oradent team, I noticed your booking form..." | wa.me/923249...?text=... |

Each row has a **WhatsApp Send Link** — click it, review the message, and hit send.

---

## Business Terms Glossary

| Term | Meaning |
|---|---|
| **B2B** | Business to Business — selling to businesses not individuals |
| **Lead** | A potential customer who might buy from you |
| **Lead Generation** | The process of finding potential customers |
| **Outreach** | The first message you send to a potential customer |
| **Sales Pitch** | A message designed to start a sales conversation |
| **CRM** | Customer Relationship Manager — organised storage of leads |
| **Latent Value** | Hidden revenue a business is losing without realising |
| **Conversion Rate** | Percentage of leads that become paying customers |
| **HITL** | Human in the Loop — human reviews AI output before sending |
| **Automation Gap** | The difference between what a business does vs what it could do with technology |

---

## Adapting This for Other Niches

This pipeline is fully reusable. To target a different industry or city:

1. Change the Scout query:
```json
{"q": "GP Clinics in London", "limit": 20}
```

2. Update the Chatbot Detector conditions to match relevant tools for that industry

3. Update the AI Agent prompt to match your new product and value proposition

That's it. Everything else stays the same.
