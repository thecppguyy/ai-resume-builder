# AI Resume Builder V1 — Implementation Reference

## Summary

Build an English-language resume editor for recent graduates. Visitors can draft and preview a resume before creating an account. Signed-in users can save it, get AI writing help, tailor it to a job description, review feedback, and download a PDF. Pro users can also export DOCX and keep multiple resumes.

Keep the chosen stack: the existing Next.js App Router project, Neon Postgres, Neon Auth, Neon Object Storage, OpenAI, Stripe, and the existing Cloudways server. The repository is a clean starter pinned to Next.js 16.4.2 and React 19.3.0. The installed Next.js package requires Node 20.9 or newer; the stated Cloudways Node 22+ runtime meets that minimum. Next.js supports self-hosting with `next start`; confirm the Cloudways proxy passes streaming responses and that private routes are not cached. See [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting).

Neon’s current backend includes Managed Better Auth and S3-compatible Object Storage. Use the current Neon Auth SDK and the configuration documented for the project when implementation begins. See [Neon Auth](https://neon.com/blog/neon-auth-branchable-identity-in-your-database) and [Neon Object Storage](https://neon.com/blog/building-neon-object-storage).

## Product behavior

### Visitor and account flows

- Public landing and pricing pages link to Privacy and Terms pages. Index public marketing pages; exclude sign-in, dashboard, editor, and resume data from search indexing.
- A visitor chooses a blank resume or a sample. The editor, template choice, preview, and basic feedback checklist work before signup.
- Save guest edits in browser storage. Require an account for AI actions, job-match analysis, and downloads. On signup, import the guest draft automatically; clear the browser copy only after the database save succeeds.
- If a guest draft cannot fit the account’s Free resume limit, keep it locally and ask the user to choose which resume to keep or upgrade. Never silently overwrite saved work.
- Use Neon Auth for Google OAuth and email magic links. Use Resend for auth email delivery. Configure trusted origins and provider callbacks for local and production domains.
- Signed-in users land on a dashboard showing saved resumes, plan status, and a create-resume action. The editor autosaves and shows saving, saved, or error status.
- On autosave failure, retain the pending draft locally and allow retry. Use an update/version check to avoid silently overwriting a newer edit from another tab.

### Resume editor and templates

- Provide three single-column templates that share structured resume content. Templates are selectable, with no font, color, or spacing controls.
- Store fixed sections: contact details, summary, work history, education, and skills. Support repeatable work and education entries, ordered experience bullets, and optional contact fields.
- Show editor and formatted preview side by side on desktop; keep the form and preview usable on mobile.
- Allow blank or original fictional sample content. Resume importing and upload parsing are out of scope for V1.

### AI, tailoring, and feedback

- Keep OpenAI calls server-side. Send only the minimum relevant resume/job text, and only after the user explicitly requests an AI action.
- Implement summary drafting/improvement and bullet rewriting as proposals. Users can edit, accept, or reject each proposal; never replace source content automatically.
- Treat job matching as a separate monthly allowance from writing actions. Extract required/preferred terms, show present and missing terms, and suggest where a term could fit. Do not invent qualifications or experience; phrase unsupported additions as “if accurate” prompts.
- Validate structured model responses against server-side schemas before returning them. Use `gpt-6-luna` as the configurable default model; keep the model ID in server configuration. Review the current [OpenAI model catalog and pricing](https://platform.openai.com/pricing) and [Responses API structured output reference](https://platform.openai.com/docs/api-reference/responses) during implementation.
- Make general feedback deterministic so Free feedback can remain unlimited without consuming AI quota. Cover completeness, clarity, and formatting. Add job-match feedback only after a job description has been analyzed.
- AI failures, invalid output, and rate-limit errors show a retryable message and do not consume quota. Reserve quota before a provider call, then commit on success or release the reservation on failure.

### Plans, exports, and billing

| Plan | Entitlements |
|---|---|
| Free | One saved resume; one template; 10 writing AI actions per calendar month; one job-match check per calendar month; unlimited general feedback checklists; two PDF downloads per calendar month |
| Pro Monthly | $9.99/month; multiple resumes; all templates; unlimited AI writing, job tailoring, feedback, and PDF downloads; DOCX exports |
| Pro Annual | $59.99/year; same Pro entitlements |

- Provide a 7-day Pro trial without a card at trial start. Apply the trial to both Pro billing options. If no payment method is present at trial end, cancel the subscription and return the user to Free. Stripe supports cancellation when a trial ends without a payment method; see [Stripe subscription trial settings](https://docs.stripe.com/api/subscriptions/update?api-version=2025-04-30.basil).
- Use Stripe Checkout to start subscriptions and the Stripe customer portal for payment updates and cancellation. Cancellation keeps Pro entitlements through the paid period.
- Store Stripe customer/subscription identifiers and the latest entitlement state in Postgres. Process webhooks idempotently; Stripe remains the authority for billing status.
- On downgrade, keep all resumes but lock all except one selected active resume. Do not delete excess resumes. Allow access again after upgrade.
- Require an account for export. Count each requested PDF download against the Free allowance; DOCX is Pro-only. Store generated files in a private Neon bucket for 30 days. Serve downloads through short-lived signed URLs after verifying export ownership.
- Delete expired exports on a scheduled Cloudways task. Account deletion removes resumes, tailoring data, usage records, and stored objects; retry cleanup failures and do not leave private exports accessible.

## Architecture and interfaces

### Next.js boundaries

- Keep the App Router and installed Next.js version. Use Server Components for public pages and protected data reads; use Client Components only where editing, browser storage, or interactive preview requires them.
- Put database access in a server-only data-access layer. Authenticate and verify ownership on every query and mutation, including each Server Action. Do not rely only on a protected layout or hidden UI control.
- Use Server Actions for resume mutations and settings forms. Use Route Handlers for OpenAI calls, Stripe webhooks/checkout/portal, export generation/downloads, and account deletion.
- Do not cache user-specific resume, tailoring, usage, billing, or export data. Review existing `next.config.ts` flags against the installed Next.js documentation before changing them.

### Pages and routes

- Public pages: `/`, `/pricing`, `/privacy`, `/terms`, `/sign-in`
- Authenticated pages: `/dashboard`, `/resumes/new`, `/resumes/[resumeId]`, `/account/billing`, `/account/privacy`
- Server endpoints: `/api/ai/summary`, `/api/ai/rewrite-bullet`, `/api/ai/job-match`, `/api/billing/checkout`, `/api/billing/portal`, `/api/webhooks/stripe`, `/api/exports`, `/api/exports/[exportId]/download`, `/api/account/delete`

Return typed success/error results from app mutations. Return structured AI results with validated suggestion records. Stripe webhook handlers return success for already-processed event IDs and record each event once.

### Neon and application data

Use Drizzle ORM with migrations and a pooled Neon Postgres connection. Keep the managed `neon_auth` schema owned by Neon; reference its user ID from app-owned rows.

- `resumes`: owner, title, selected template, structured JSONB content, timestamps, and version.
- `job_sessions`: one row per resume, unique on `resume_id`; job description, extracted terms, match results, and latest analysis timestamp.
- `ai_suggestions`: owner/resume, operation type, target section/bullet, original content, proposed content, review status, and timestamps. Accepted content is also saved to the resume.
- `subscriptions`: owner, Stripe customer/subscription IDs, current plan/status, period/trial end, and cancellation state.
- `monthly_usage`: owner and UTC calendar month, with separate counters for writing AI actions, job-match checks, and PDF downloads. Enforce one row per owner/month and update counters transactionally.
- `exports`: owner/resume, format, private object key, creation time, and expiry time.
- `stripe_webhook_events`: unique Stripe event ID, event type, and processing timestamp for idempotency.

Validate input on the server. Enforce resume limits and monthly quotas inside transactions so concurrent requests cannot exceed them. Use foreign keys and owner-filtered queries for user-owned data.

### AI and document generation

- Use one server-side OpenAI integration with output schemas for summary proposals, bullet rewrites, term analysis, and placement suggestions. Never put the API key in a `NEXT_PUBLIC_` environment variable.
- Use semantic preview content as the shared source for on-screen and export view models.
- Generate selectable-text PDFs with a Node-compatible PDF renderer and DOCX from the same structured view model. Keep layouts single-column, text-first, and consistent in section order. Verify pagination for long resumes and long bullet lists.
- Upload completed files to a private Neon Object Storage bucket; store object keys and expiry metadata in Postgres. Do not expose bucket credentials or permanent public object URLs.

## Security, reliability, and deployment

- Keep database, OpenAI, Stripe, and object-storage credentials server-only. Validate Stripe webhook signatures and reject invalid requests.
- Apply authorization checks to every owner-scoped read/write and every download. Apply request-size limits and per-user/IP throttling to expensive AI and export routes.
- Do not log resume text, job descriptions, magic-link tokens, payment details, or signed download URLs. Log request IDs, operation names, statuses, and sanitized error details to Cloudways/Neon logs only.
- Keep AI, database, auth, and storage failures recoverable: preserve unsaved editor state, show retry controls, and do not charge quota for failed AI/export generation.
- Use local and production Neon branches with separate environment variables. Use Stripe test mode locally and live mode in production.
- Deploy from the selected Git branch to the existing Cloudways server with Node 22+. Build with `next build` and run the production Node server using the Cloudways process manager. Confirm reverse-proxy streaming, HTTPS, graceful restart, persistent build assets, and environment-variable availability before release. See [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting).
- Do not add an admin UI or product analytics service. Use provider logs for production troubleshooting.

## Implementation phases and acceptance criteria

1. **Foundation:** Replace starter content with the public shell and app navigation; connect Neon, Drizzle migrations, Neon Auth, and route protection.  
   **Accept when:** Google and magic-link sign-in work locally and in production; authenticated data is owner-scoped.

2. **Resume editor:** Implement blank/sample start, fixed sections, three templates, guest browser autosave/import, signed-in autosave, dashboard, and preview.  
   **Accept when:** A guest can create a draft, sign up, import it once, and return later to edit a saved resume without losing changes.

3. **Feedback and AI:** Add deterministic feedback checklist, AI summary and bullet proposals, and job description analysis with review controls and quotas.  
   **Accept when:** Suggestions are editable and individually accepted/rejected; missing terms do not create unsupported experience; quota races and provider failures behave correctly.

4. **Export and storage:** Generate PDF and DOCX, upload private objects, issue owner-checked signed downloads, and add expiry/account-deletion cleanup.  
   **Accept when:** PDF and DOCX are readable and text-selectable, Free/Pro export limits are enforced, expired files are removed, and account deletion removes related data and objects.

5. **Billing and launch:** Add pricing page, Stripe Checkout, trial handling, webhooks, customer portal, downgrade behavior, legal pages, and Cloudways deployment.  
   **Accept when:** Test-mode monthly/annual checkout, trial cancellation, renewal, cancellation-at-period-end, downgrade locking, and webhook retries produce correct entitlements.

Release readiness means the full user journey works: start a resume, sign up, edit and save, tailor it to a job, review feedback, upgrade where needed, and download the allowed format. Do not treat an unverified Cloudways deployment or failed privacy/deletion behavior as release-ready.

## Test plan

- **Unit:** resume validation, content normalization, template mapping, deterministic feedback rules, quota period calculation, entitlement checks, and AI output validation.
- **Integration:** owner isolation, resume create/update/delete, one-session-per-resume constraint, concurrent quota requests, object upload/sign/delete, account-deletion retries, and idempotent Stripe webhook processing.
- **End-to-end:** guest draft import; blank and sample resumes; AI suggestion accept/reject; job-match quota; Free PDF limits; Pro DOCX and multiple resumes; trial without card; cancellation at period end; downgrade locking; mobile editing; slow network and failed autosave recovery.
- **Export checks:** all three templates, page breaks, long content, Unicode characters, selectable PDF text, DOCX reopening, expired export denial, and cross-account download denial.
- **Deployment smoke checks:** production build/start, OAuth and email callback domains, Stripe webhook signature, private response caching, HTTPS download links, and process restart behavior.

## Assumptions and open risks

- Monthly quota boundaries use UTC calendar months. Each requested PDF download consumes a Free allowance, including another download of a stored export.
- Guests can edit, preview, and run basic feedback; account creation is required for AI, job matching, and exports.
- Trial applies to both Pro options; if no card is added by trial end, Stripe cancels it and the user returns to Free.
- Brand styling was not specified; use an accessible, restrained light theme and the repo’s existing styling setup.
- Privacy and Terms pages require owner/legal review. They must explain OpenAI processing and export retention.
- Confirm Cloudways process-manager/proxy details, OAuth domains, Resend domain verification, and Neon region availability before launch.
- Review the chosen OpenAI model on representative graduate resumes and job descriptions before launch; change the configured model if results are inadequate.
- Verify account deletion and scheduled object cleanup against actual Neon and Cloudways behavior before promising completion to users.
