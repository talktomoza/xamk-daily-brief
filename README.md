# 🌙 Xamk Daily Brief

A personal AI study assistant that checks **Xamk Learn** (Moodle) every evening and sends a short study report in **Bangla** by email. Students can use it to plan the next day.

> প্রতিদিন রাতে Xamk Learn চেক করে বাংলায় একটা ছোট স্টাডি রিপোর্ট ইমেইলে পাঠায়, যাতে পরের দিনের প্ল্যান করা সহজ হয়।

Built for a working master's student (Digital Industry and AI, Xamk, Finland) who also does shift work and has limited study time.

---

## ✨ What it does

- Runs automatically every night at about **21:40 Finnish time** (Europe/Helsinki)
- Opens Xamk Learn in a browser where the student is already logged in
- Collects:
  - 🔴 Overdue assignments and anything due in the next 3 days
  - 📚 Remaining assignments (course, title, deadline)
  - ✅ Submission status (submitted / not submitted / submissions closed)
  - 🗓️ Classes, meetings and events for tomorrow and the rest of the week
- Writes a short, friendly report in Bangla with a 3-step plan for tomorrow
- Sends the report by **Gmail** to the student's own address and pushes a **phone notification**

## 🧱 How it works

```
Scheduled task (21:40 EET)
        │
        ▼
Claude (Cowork) ──► Built-in browser on the student's computer
        │               ├─ learn.xamk.fi/my/                 (Dashboard / Timeline)
        │               ├─ learn.xamk.fi/calendar/view.php?view=upcoming
        │               └─ Moodle AJAX (read-only):
        │                    core_calendar_get_action_events_by_timesort
        │
        │   Fallback if the browser is offline: personal iCal export feed
        ▼
Bangla report ──► Gmail (send_message, to self only)
              └─► Push notification
```

| Part | Tool |
|---|---|
| Scheduling | Claude Cowork scheduled task |
| Reading Learn | Claude desktop app built-in browser (`get_page_text`, read-only JavaScript `fetch`) |
| Fallback data | Moodle calendar iCal export URL |
| Delivery | Gmail connector + push notification |

### Why the Moodle AJAX call?

The Timeline block on the Dashboard loads with JavaScript, so its text is not always visible to a page reader. The same data comes from Moodle's own read-only web service, which the logged-in page already uses:

```js
const now = Math.floor(Date.now() / 1000);
const res = await fetch(
  '/lib/ajax/service.php?sesskey=' + M.cfg.sesskey,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify([{
      index: 0,
      methodname: 'core_calendar_get_action_events_by_timesort',
      args: {
        limitnum: 50,
        timesortfrom: now - 30 * 86400,   // look back 30 days for overdue items
        timesortto:   now + 60 * 86400,   // look ahead 60 days
        limittononsuspendedevents: true
      }
    }])
  }
).then(r => r.json());
```

Useful fields in each event: `name`, `course.fullname`, `timesort`, `overdue`, `action.name` and `action.actionable`. When `actionable` is `false`, Learn no longer accepts the submission (or hasn't opened it yet).

## 📝 Report format (Bangla)

```
শুভ সন্ধ্যা [নাম] বন্ধু! 🌙 আজ [তারিখ, বার]

🔴 জরুরি (দেরি হয়ে গেছে / ৩ দিনের মধ্যে)
- ...

📚 বাকি অ্যাসাইনমেন্ট
- [কোর্স] – [অ্যাসাইনমেন্ট] – ডেডলাইন: [তারিখ, সময়]

🗓️ আগামীকাল ও এই সপ্তাহের ক্লাস/ইভেন্ট
- ...

✅ আগামীকালের প্ল্যান
1. ...
2. ...
3. ...

💪 [ছোট একটা উৎসাহের কথা]
```

## 🚀 Setup

1. Install the **Claude desktop app** and log in to Xamk Learn once in its **built-in browser**. The sign-in is saved.
2. Connect the **Gmail** connector in Claude.
3. Create a **scheduled task** in Cowork:
   - Schedule: every day, about 21:40 Europe/Helsinki (`CRON_TZ=Europe/Helsinki 40 21 * * *`)
   - Prompt: copy [`prompt-template.md`](#-prompt-template) below and fill in your own details
4. Optional: in Learn, go to **Calendar → Import or export calendars → Export** to get your personal iCal URL for the fallback.
5. Keep the computer on and the Claude app open at the scheduled time.

## 📋 Prompt template

```text
You are "Xamk Daily Brief", <NAME>'s personal study assistant. Make the evening
study report now (it runs every night around 21:40 Finnish time).

ABOUT THE STUDENT
- Name: <NAME>. Call them "<NICKNAME>".
- Master's student at Xamk. Study time is limited.
- Write the whole report in Bangla. Short, friendly, clear.

WHAT TO DO
1. Use the built-in browser (load the built-in-browser skill first).
   The student is already logged in to Xamk Learn there.
2. Read https://learn.xamk.fi/my/ and
   https://learn.xamk.fi/calendar/view.php?view=upcoming with get_page_text.
   Open course/assignment pages only if needed to confirm deadlines or status.
3. If the browser is unreachable, use this iCal feed instead: <YOUR_ICAL_URL>
4. Collect: assignments due, overdue items, items due within 3 days,
   classes/events tomorrow and this week.

RULES
- READ ONLY in Learn. Never submit, upload, delete, change or post anything.
- Only report what you actually found. Never invent assignments or dates.
- Use Finnish time (Europe/Helsinki).
- If Learn cannot be accessed, still send the email and say so clearly.

DELIVERY
1. Email the report with Gmail (send_message) to <YOUR_EMAIL> only.
   Subject: Xamk Daily Brief – [date]
2. Also send it with SendUserMessage and a short push notification.
```

## 🔒 Safety and privacy

- **Read-only:** the assistant never submits, uploads, deletes or posts anything in Learn.
- **No made-up data:** it only reports assignments and dates it actually found.
- **Email goes to yourself only.**
- **⚠️ Never commit your iCal URL.** It contains a `userid` and a private `authtoken` that give anyone access to your calendar. Keep it in the scheduled task prompt only. If it leaks, reset it in Learn (**Calendar → Export → Get calendar URL** creates a new token).
- Do not commit your email address, student ID or screenshots of personal grades.

## ⚠️ Limitations

- The computer and the Claude desktop app must be on at run time; otherwise only the iCal fallback works.
- Learn may not respond during maintenance windows (for example 07:00–08:00).
- Group work and forum activities may show partial status. For example, a forum can say *"Start discussions: 1 to do, Post replies: 2 done"*.
- Moodle's Timeline only shows items that still need action, so already submitted work may appear only in the calendar.

## 🗺️ Ideas for later

- [ ] Weekly summary on Sunday
- [ ] Add shift-work schedule so the plan fits real free time
- [ ] Reminder 24 h before each deadline
- [ ] English / Finnish report option

## 📄 License

MIT. Free to use and adapt for your own studies.
