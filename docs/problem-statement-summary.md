# CertiFlow -- Problem Statement Summary

## 1. Project overview

**CertiFlow: Skilling Certificate Issuance Platform** is a platform for
the fictional Pragati Skills Mission. It is intended to manage the
complete process of certifying candidates trained by private training
centres: batch submission, multi-level review and approval, certificate
generation and delivery, and public authenticity verification.

The programme trains approximately **8,000 people per month** through
more than **300 private training centres**.

## 2. Current problem

The existing process is manual: - Training centres email spreadsheets of
passed candidates to district offices. - District officers approve
batches through WhatsApp. - Clerks manually create certificates in Word
and email them. - A batch can take weeks to reach candidates. - There is
no reliable audit trail of who approved what. - Edited or fake
certificates cannot be checked by employers.

The programme wants certificates delivered to candidates' phones within
days, traceable decisions, and public verification within seconds.

## 3. Users and roles

  -----------------------------------------------------------------------
  User / role                         Main responsibilities
  ----------------------------------- -----------------------------------
  Training centre (demo user: Anika)  Create batches, upload candidate
                                      data, submit batches, track
                                      progress, correct returned batches
                                      and resubmit

  District officers (demo users: Dev  Review batches assigned to their
  and Meera)                          district; approve, reject or return
                                      with a reason. A district decision
                                      requires more than one officer to
                                      agree

  Quality team (demo user: Neha)      Perform additional scrutiny of
                                      large batches

  State office (demo user: Farah)     Make the final approval decision,
                                      monitor stuck work and cancel
                                      certificates that should not have
                                      been issued

  Candidates                          Receive certificates through
                                      WhatsApp and download them

  Employers / public                  Verify whether a certificate is
                                      genuine without logging in
  -----------------------------------------------------------------------

## 4. Required capabilities

1.  **Batch management**
    -   Training centres create batches and add candidates from existing
        spreadsheets.
    -   Submit batches for approval and track their status.
    -   Correct and resubmit batches when they are returned.
2.  **Review and approval**
    -   District officers review batches and approve, reject or return
        them with a reason.
    -   More than one district officer must agree.
    -   Large batches receive additional quality scrutiny.
    -   The state office makes the final decision.
3.  **Certificate issuance and delivery**
    -   After approval, issue certificates for candidates.
    -   Deliver each candidate's certificate through WhatsApp.
    -   Candidates must be able to download their certificates.
    -   The WhatsApp integration is a stand-in provider that must
        simulate failures and timeouts; no real WhatsApp account is
        provided.
4.  **Correction and cancellation**
    -   Officers must be able to correct mistakes through the defined
        workflow.
    -   The state office can cancel certificates that should not have
        been issued.
5.  **Public verification**
    -   Anyone can verify a certificate without logging in.
    -   A cancelled certificate must no longer appear genuine.
6.  **Live updates and monitoring**
    -   Users should see changes as they happen without refreshing.
    -   Stuck batches must be identified and escalated to someone
        senior.

## 5. Critical scenarios that must work

1.  A spreadsheet contains invalid rows: show exactly what is wrong and
    save none of the batch if validation fails.
2.  Two district officers approve simultaneously: the batch must advance
    only once.
3.  A batch of 60 candidates is approved: each candidate must have
    exactly one certificate, even if WhatsApp is unreliable.
4.  The state office cancels a fake certificate: a public check
    performed afterward must report it as invalid.
5.  A batch remains untouched with an officer too long: a senior user
    must be alerted without manual chasing.
6.  A batch is returned and resubmitted: the system must have a
    deliberate, documented policy for what happens to earlier approvals.

## 6. Rules that must never be violated

-   A user cannot perform an action belonging to another workflow step
    or act as another person.
-   A candidate must not receive two certificates for the same
    certification.
-   Two certificates must not share a certificate number.
-   A cancelled certificate must never verify as genuine.
-   A stranger must not access another person's certificate or personal
    details.
-   Every decision must be traceable to the person who made it and when.

## 7. Demo identities

The reviewers should be able to switch between these sample users and
try the flows: - Anika -- training centre - Dev -- district officer -
Meera -- district officer - Neha -- quality team - Farah -- state office

## 8. Mandatory technical expectations

-   **Event-driven:** loosely coupled components communicate through
    events; slow or unreliable work must not block the user.
-   **Scalable:** add capacity horizontally and preserve correctness
    through instance restarts.
-   **Containerised:** the complete system runs in containers and starts
    locally with one command.
-   **Tested:** automated tests prove the critical scenarios and
    invariants.
-   **Secure:** enforce role and data access; reject tampered requests
    and invalid input.
-   **Observable:** trace what happened to any batch or certificate end
    to end.

## 9. Required technology stack

  -----------------------------------------------------------------------
  Layer                               Required technology
  ----------------------------------- -----------------------------------
  Frontend                            React with Vite

  Backend                             Node.js with Express and TypeScript

  Database                            PostgreSQL with Prisma or Drizzle
                                      and proper migrations

  Caching                             Redis

  Background jobs / events            A queue or message broker of the
                                      developer's choice, with the choice
                                      explained in README
  -----------------------------------------------------------------------

The brief does not mandate a specific queue or broker.

## 10. Local execution and deployment plan

-   A fresh clone must be runnable locally by following the setup guide.
-   Demo data must be available.
-   Automated tests must be runnable.
-   Public deployment is not required.
-   A `DEPLOYMENT.md` must describe a production plan for:
    -   100,000 registered users and around 10,000 daily active users.
    -   Peak of around 500 batches approved in one hour, generating
        around 30,000 PDFs and sending around 30,000 WhatsApp messages.
    -   Public verification receiving thousands of lookups daily in
        bursts.

The deployment plan must cover architecture and event flow, hosting
choices, horizontal scaling and throughput limits, backlog recovery,
worker crashes, Redis restarts, WhatsApp outages, slow database
behaviour, certificate storage/access control, monitoring/alerting,
estimated monthly cost and trade-offs.

## 11. Submission requirements and deadline

Submit through the form specified in the problem statement:
https://forms.gle/rWx1qemA5RGPSMq89

Required deliverables by **2:30 pm**: 1. Public GitHub repository
containing the project, Dockerfiles, Docker Compose and automated tests.
2. README with requirements note first, system architecture and key
technical decisions with reasons. 3. Local setup guide that takes a
fresh clone to a running system with demo data and passing tests. 4.
`DEPLOYMENT.md` covering the production plan. 5. Demo video of **2
minutes or less**, shared through Google Drive with "Anyone with the
link" access or Loom.

The deadline is final; no extensions are offered.

## 12. Evaluation criteria

The reviewers assess: - **Problem understanding:** requirements note and
decisions made where the brief is unclear. - **Correctness:** critical
scenarios work and prohibited outcomes never occur. - **Code quality:**
structure, naming, readability, meaningful commits and useful README. -
**Working demo:** demonstrated functionality actually runs. -
**Architecture and technology choices:** component interactions, failure
handling and scale. - **Tests:** tests prove the important scenarios and
rules. - **AI usage:** where AI was used and how its output was
checked. - **Self-awareness:** honest known bugs, limitations and next
steps.

## 13. Explicitly unspecified details

The problem statement does not define: - Exact CSV/Excel column names or
file format. - The numerical threshold for a "large batch". - The exact
number of district approvals beyond requiring more than one officer. -
Detailed certificate layout or numbering format. - Exact SLA duration
for a batch to be considered stuck. - WhatsApp provider API contract. -
Exact batch state names or the policy for earlier approvals after
resubmission.

These must be documented as implementation decisions rather than
represented as requirements from the brief.
