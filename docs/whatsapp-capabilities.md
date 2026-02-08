# WhatsApp Business API Capabilities

> Reference document for Jadwala's WhatsApp integration

## Overview

WhatsApp Business API (Cloud API) provides the messaging infrastructure needed for healthcare scheduling. This document outlines capabilities, limitations, and implementation considerations.

---

## Message Types

### 1. Template Messages (Outbound)
Pre-approved messages for initiating conversations. Required for any message sent outside the 24-hour session window.

**Use Cases:**
- Appointment reminders
- Confirmation requests
- Cancellation notices
- Follow-up messages

**Structure:**
```
Header (optional): Text, Image, Video, or Document
Body: Text with variables {{1}}, {{2}}, etc.
Footer (optional): Text
Buttons (optional): Quick Reply or CTA
```

**Example:**
```
مرحباً {{1}}،

موعدك مع {{2}} يوم {{3}} الساعة {{4}}.

هل تؤكد الحضور؟

[تأكيد] [إعادة جدولة] [إلغاء]
```

**Approval:**
- Submitted via Meta Business Manager
- Review takes 24-48 hours
- Must comply with commerce & healthcare policies

---

### 2. Session Messages (Interactive)
Free-form messages within 24-hour window after user reply.

**Interactive Components:**

#### Quick Reply Buttons
- Max 3 buttons
- 20 characters per button
- Returns button payload on click

```json
{
  "type": "button",
  "body": { "text": "Choose an action:" },
  "action": {
    "buttons": [
      { "type": "reply", "reply": { "id": "confirm", "title": "✓ Confirm" }},
      { "type": "reply", "reply": { "id": "reschedule", "title": "📅 Reschedule" }},
      { "type": "reply", "reply": { "id": "cancel", "title": "✗ Cancel" }}
    ]
  }
}
```

#### List Messages
- Up to 10 options in sections
- Good for time slot selection

```json
{
  "type": "list",
  "body": { "text": "Select a new appointment time:" },
  "action": {
    "button": "View Available Slots",
    "sections": [
      {
        "title": "Wednesday, Feb 12",
        "rows": [
          { "id": "wed_10am", "title": "10:00 AM", "description": "Dr. Ahmed" },
          { "id": "wed_2pm", "title": "2:00 PM", "description": "Dr. Ahmed" }
        ]
      },
      {
        "title": "Thursday, Feb 13",
        "rows": [
          { "id": "thu_11am", "title": "11:00 AM", "description": "Dr. Ahmed" }
        ]
      }
    ]
  }
}
```

#### CTA Buttons
- Up to 2 buttons
- Types: URL or Phone

```json
{
  "type": "button",
  "body": { "text": "Complete your deposit to confirm:" },
  "action": {
    "buttons": [
      { "type": "url", "url": "https://pay.jadwala.com/{{session_id}}", "title": "Pay Now" },
      { "type": "phone_number", "phone_number": "+966500000000", "title": "Call Clinic" }
    ]
  }
}
```

---

### 3. Location Messages

**Sending Location:**
```json
{
  "type": "location",
  "location": {
    "longitude": 46.6753,
    "latitude": 24.7136,
    "name": "King Faisal Specialist Hospital",
    "address": "Al Maather, Riyadh"
  }
}
```

**Receiving Location:** Users can share their location for home visits.

---

## Webhooks & Events

### Inbound Events
- `messages` — New message received
- `statuses` — Delivery/read receipts

### Status Tracking
```
sent → delivered → read
```

**Payload Example:**
```json
{
  "statuses": [{
    "id": "wamid.xxx",
    "status": "read",
    "timestamp": "1707388800",
    "recipient_id": "966500000000"
  }]
}
```

---

## Rate Limits & Tiers

### Quality Rating
Based on user feedback (blocks, reports).

| Tier | Daily Template Limit |
|------|---------------------|
| Unverified | 250 |
| Tier 1 | 1,000 |
| Tier 2 | 10,000 |
| Tier 3 | 100,000 |
| Tier 4 | Unlimited |

**Upgrade:** Automatic based on volume + quality over 7 days.

---

## Payments

### Native WhatsApp Pay
- Available: India, Brazil
- **Not available in Saudi Arabia**

### Workaround for MENA
1. Send CTA button with payment link
2. User completes payment on external gateway (Tap, HyperPay, Moyasar)
3. Webhook confirms payment
4. Send confirmation message

```
Payment Flow:
[WhatsApp] → [Payment Link] → [Gateway] → [Webhook] → [WhatsApp Confirmation]
```

---

## Healthcare-Specific Considerations

### HIPAA/Data Privacy
- Cloud API: Data processed by Meta (US)
- On-Premise API: Self-hosted, full control
- **Recommendation for Saudi:** Cloud API acceptable with proper DPA; On-Premise for sensitive cases

### NDMO Compliance
- Ensure data residency requirements are met
- Cloud API stores message content for 30 days
- On-Premise gives full control

### Template Content Rules
- No promotional content in appointment messages
- Medical information must be factual
- Include opt-out mechanism

---

## Technical Architecture

### Cloud API (Recommended for MVP)
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Jadwala   │────▶│  Meta Cloud │────▶│  WhatsApp   │
│   Backend   │◀────│     API     │◀────│    User     │
└─────────────┘     └─────────────┘     └─────────────┘
      │
      ▼
┌─────────────┐
│  Hospital   │
│    EHR      │
└─────────────┘
```

### On-Premise API (Enterprise)
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Jadwala   │────▶│  On-Premise │────▶│  WhatsApp   │
│   Backend   │◀────│   Docker    │◀────│    User     │
└─────────────┘     └─────────────┘     └─────────────┘
                          │
                    (Self-hosted)
```

---

## Limitations & Workarounds

| Limitation | Workaround |
|------------|------------|
| No native date picker | List message with time slots |
| No forms | Multi-step conversational flow |
| No payments (Saudi) | External payment link + webhook |
| Template approval delay | Pre-approve common templates |
| 24-hour session window | Use templates to re-engage |
| 3 button max | Use list messages for more options |

---

## Sample Flows

### Appointment Confirmation
```
1. [Template] Reminder sent 24h before
2. [User] Taps "Confirm"
3. [Session] "Great! See you tomorrow at 10 AM. Here's the location:"
4. [Location] Clinic map
```

### Rescheduling
```
1. [User] Taps "Reschedule"
2. [Session] List message with available slots
3. [User] Selects new slot
4. [Session] "Confirmed for Wed 2 PM. Deposit required:"
5. [CTA] Payment link
6. [Webhook] Payment confirmed
7. [Session] "All set! New appointment confirmed ✓"
```

### No-Show Follow-up
```
1. [Template] "We missed you today. Would you like to rebook?"
2. [User] Taps "Yes"
3. [Session] List of available slots
4. [Policy] If repeat no-show, require deposit
```

---

## Costs

### Cloud API Pricing (Meta)
- **Conversation-based:** Per 24-hour session
- **User-initiated:** ~$0.005 - $0.02 (varies by country)
- **Business-initiated:** ~$0.02 - $0.05

### BSP Fees (if using partner)
- Setup: $0 - $500
- Monthly: $0 - $200
- Per message: $0.001 - $0.01 additional

### Estimated Monthly Cost
| Volume | Estimated Cost |
|--------|---------------|
| 1,000 appointments | $50 - $150 |
| 10,000 appointments | $300 - $800 |
| 100,000 appointments | $2,000 - $5,000 |

---

## Implementation Checklist

- [ ] Create Meta Business Account
- [ ] Verify business
- [ ] Set up WhatsApp Business Account
- [ ] Get phone number approved
- [ ] Create message templates
- [ ] Set up webhook endpoint
- [ ] Implement message handling logic
- [ ] Test in sandbox
- [ ] Go live

---

## Resources

- [WhatsApp Cloud API Docs](https://developers.facebook.com/docs/whatsapp/cloud-api)
- [Message Templates](https://developers.facebook.com/docs/whatsapp/message-templates)
- [Interactive Messages](https://developers.facebook.com/docs/whatsapp/guides/interactive-messages)
- [Webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks)
- [Pricing](https://developers.facebook.com/docs/whatsapp/pricing)

---

*Last updated: February 2026*
