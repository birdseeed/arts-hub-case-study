# The Arts Hub operations portal

The Arts Hub is my wife's performing arts school on the Isle of Man, and since July 2026 I have built and run the portal it uses to manage classes, families, staff and money. It is taking over from the commercial membership system the school used before, and it already saves us hours of administration every month. The source stays private because it holds children's and families' details, payroll and bank information, so this is a description of how it works and how I run it rather than the code itself.

## What it does

The portal has three front doors. Staff use it for registers, timetables, their own calendars, timesheets, invoices and expense claims, and administrators manage classes, rooms, private lessons, family records, customer invoicing, payroll and the year end accountant pack. Families get their own portal showing their children's classes, invoices and statements, and new families apply through a public registration form. It takes payments through GoCardless and SumUp, sends email through Google Workspace and can send SMS through Twilio.

Before moving the whole timetable across, I ran it as a contained production pilot on two summer schools in July and August 2026, with the old system kept as the reference until a go or no go checklist had been passed.

## How it is hosted

It runs on a single Ubuntu VPS as a Laravel application on PHP 8.3 with SQLite, behind Caddy listening only on loopback. Nothing on the server is open to the internet for the application; all traffic comes in through a Cloudflare Tunnel, so the server makes an outbound connection to Cloudflare and nothing needs to reach it directly. Each hostname has its own job: the staff site is behind Cloudflare Access with email one time codes, and the application validates the Access token itself rather than trusting a header, while the family portal and the public registration form use the application's own magic link sign in and host based routing so each hostname can only reach the routes meant for it.

Releases are pulled rather than pushed. A systemd timer on the server checks the main branch on GitHub, and when a new commit appears it runs the documented deploy locally, preserving the environment file, the database and private uploads. That keeps the server outbound only, with no deploy credentials stored anywhere that can reach into it. Changes reach main through pull requests, and the test suite covers the payment, payroll, permission and family portal workflows.

## Protecting the data

Sensitive fields are encrypted at rest in the database, including addresses, phone numbers, bank details, National Insurance numbers, tax codes, and children's dates of birth, medical notes and emergency contacts. Teaching notes and staff messages are encrypted too, and audit records note who did what without storing the content. Payments to staff follow a separation of duties, so the person who approves a timesheet or expense claim cannot be the same person who records it as paid.

## Backups, monitoring and recovery

There are two independent backup layers. The hosting provider keeps seven daily full disk images, and Restic takes encrypted, application consistent backups every four hours to a private, EU based Cloudflare R2 bucket, using a token that can only touch that one bucket. The SQLite database is copied with SQLite's online backup API and integrity checked before Restic reads it, so backups never need the application to stop. The off site backups include the server's Caddy, tunnel, SSH, firewall and systemd configuration as well as the application, so the host can be rebuilt rather than just the data restored.

The recovery objectives are four hours of data loss at most for the portal and a four hour rebuild onto a clean server, with a target of one hour once the rebuild is automated. I am honest with myself that the server and tunnel connector are single instances, so those are objectives rather than guarantees. I also wrote down the gaps, such as the encryption key rotation runbook that exists but has never been exercised end to end.

Systemd timers run a health monitor that checks the public endpoints, the backups, a weekly backup maintenance job, and a monthly maintenance window on the first Sunday that patches the server, reboots it, and then checks that the web server, PHP, the tunnel and SSH access all came back before it reports success. One September run reported a failure for three minutes even though SSH was never lost, because SSH on Ubuntu is socket activated and the service looks inactive between connections, so I changed it to check for the listening socket instead.

## How I built it

I am the only developer, and the portal was built with AI assistance across more than 650 commits since July. I worked out what the school needed with my wife, made the architecture, security and data decisions, and decided what was ready to go live. A second model reviewed changes for correctness and security, and I ran a dedicated security audit of the whole application in August 2026.
