# Mailflare fork: free-tier deployment and readiness

Fork: https://github.com/arsalan1099-sudo/mailflare  
Reviewed: 2026-11-02. Source baseline: Mailflare v0.5.4, `main` at `be89083`.

> This is a configuration and evidence checklist, **not** a claim that a live mail service is deployed or production ready. Do not put API keys, passwords, or customer messages into the repository.

## Recommended initial deployment: Cloudflare Workers Free

Mailflare's Worker configuration uses D1, R2, Durable Objects and Queues. These products have free allowances, but usage limits apply. Workers Free currently supports Queues, and Cloudflare Email Routing is free. Cloudflare's own Email Sending to arbitrary recipients needs Workers Paid. To avoid that cost during initial tests, use **Resend Free** for outbound delivery (at the time of review: 3,000 emails/month and 100/day). Provider, account, and domain verification requirements still apply.

Do **not** use Render Free with the default SQLite + local blob configuration for real mail: Render Free has ephemeral storage, idles after 15 minutes, and restricts SMTP ports. A preview on Render is not a safe mail storage deployment.

## Before deploying

1. Inspect `README.md`, `docs/deployment.md`, `docs/providers.md`, `wrangler.jsonc`, and the results of the GitHub checks from the associated PR.
2. Use the **fork** deployment link in this README. The Worker name must remain **`mailflare`** because `wrangler.jsonc` has a service self-reference to that name. Branding is separate from Worker identity.
3. Select the Cloudflare account that actually owns your DNS zone and ensure that you control the email domain. A domain is needed for a custom email address; a workers.dev subdomain does not replace mail domain verification.
4. Provision or allow the installer to create a D1 database named `mailflare`, R2 Standard bucket `mailflare-raw`, and queues `mailflare-inbound`, `mailflare-outbound`, and `mailflare-agent`. Do not commit an account-specific D1 `database_id`.
5. Use **two different least-privilege tokens**, supplied in private setup forms: the deployment token for Wrangler/Cloudflare deployment, and a runtime `CF_TOKEN` limited to the DNS and routing operations you need. Read `docs/deployment.md` for the exact current permissions. Never put either token in commits, PR comments, or screenshots.
6. Keep `GITHUB_UPDATE_TOKEN` and `MAILFLARE_UPDATE_TOKEN` unset for now. The upstream `.github/workflows/deploy-update.yml` **replaces the installation branch's entire tracked tree** with upstream files; local customizations can disappear.
7. Set up sending with Resend via **Admin → Domains → Setup sending email** and verify domain ownership, SPF, DKIM and DMARC. Mailflare's own Cloudflare Email Sending is not an arbitrary-recipient free-plan alternative.
8. Consider optional Turnstile and the application's MFA before adding real users. Keep mail/backup data private and configure monitoring and an independent backup strategy.

## First-run sequence

1. Deploy from the **fork**, not the upstream repository.
2. Open the generated `https://<worker-subdomain>.workers.dev/setup` (use the actual URL displayed by Cloudflare).
3. Allow first-run initialization to apply committed migrations, then create a strong admin account. Do not run remote migrations manually unless the setup/recovery guide explicitly requires that.
4. Add the domain in **Admin → Domains**. Choose Cloudflare Email Routing for receiving and Resend for sending; the product supports separate providers per domain.
5. Review DNS changes **before confirming** any operation that replaces existing MX records, as that can redirect or interrupt mail delivery.
6. Create a first mailbox only after confirming the DNS plan. Send mail from a **separate external mailbox** to the new address. Verify the message appears exactly once and its attachment, if any, is intact.
7. Send a reply to the external mailbox and confirm delivery plus SPF/DKIM/DMARC results in the received headers. Test login/logout, an invalid password, reset email, and at least one mailbox permission boundary.
8. Verify `/api/setup/status` returns a valid response and inspect Cloudflare Worker logs and mail queues for errors. Run backups and confirm you can recover them before relying on the service.

## Security and readiness notes

- `next.config.ts` currently has `typescript.ignoreBuildErrors: true`: **a successful bundle does not prove type correctness**. Add a strict typecheck gate once upstream build compatibility is established.
- The source includes security measures (CSP headers, authenticated real-time sockets, inbound message validation, webhook SSRF restrictions, attachment warnings) and security tests. **These are code-review observations, not penetration-test results.**
- `docs/security.md` explains that deleting an attachment does not erase the bytes from a saved original MIME message. Mailflare does not have a full antivirus engine.
- `src/lib/licenses/service.ts` and `src/lib/branding/service.ts` confirm admin branding customization is **Pro/Team gated**. Do not circumvent license enforcement as part of this free deployment plan.
- Mailflare includes an AI assistant, but AI provider and usage costs are separate; the core mail system should be tested with AI disabled.
- The source is licensed under the terms in `LICENSE` (AGPL-3.0). Preserve notices and meet source-availability requirements if you modify and serve it.
- Cloudflare free-tier usage is bounded: D1, R2, Workers, Queues and Durable Objects can reach limits. Regularly review account usage and alerts; `$0` is not an unlimited-service promise.

## Local quality checks

Node.js 22 and npm:

```bash
npm ci
npm run lint
node --test tests/*.test.mjs
npm run build
```

The attached GitHub Actions workflow attempts these checks, but the results must be reviewed before any release. This fork audit did not execute a local build because direct dependency/source access from the audit runtime was unavailable.

## Official plan/pricing verification links

- Cloudflare Workers: https://developers.cloudflare.com/workers/platform/limits/
- Cloudflare Email Sending: https://developers.cloudflare.com/email-service/platform/pricing/
- Cloudflare Queues free-plan launch: https://developers.cloudflare.com/changelog/post/2026-02-04-queues-free-plan/
- Cloudflare D1 limits: https://developers.cloudflare.com/d1/platform/limits/
- Cloudflare R2: https://developers.cloudflare.com/r2/pricing/
- Resend: https://resend.com/pricing
- Render Free limitations: https://render.com/docs/free
