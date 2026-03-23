# Communications Hub Architecture

## Key Files
- `src/components/communications/communications-inbox.tsx` — Main inbox layout (folders, search, compose button, accounts)
- `src/components/communications/thread-detail.tsx` — Thread viewer (messages, reply, forward, Reply All, attachments)
- `src/components/communications/compose-email-modal.tsx` — Outlook-style compose modal
- `src/components/communications/rich-text-editor.tsx` — TipTap editor wrapper
- `src/components/communications/template-picker.tsx` — Email template selector
- `src/components/communications/attachment-list.tsx` — Attachment display/download
- `src/lib/microsoft/outlook.ts` — Graph API email functions (send, reply, forward, fetch)
- `src/lib/microsoft/graph-client.ts` — Graph API client with token refresh

## API Routes
- `POST /api/communications/compose` — Send new email
- `GET/POST /api/communications/threads/[id]/messages` — List/send reply
- `POST /api/communications/threads/[id]/messages/[msgId]/forward` — Forward
- `GET /api/communications/threads/[id]/messages/[msgId]/attachments` — Attachments
- `POST /api/communications/tickets/[id]/comments` — Ticket comments
- `GET/POST /api/communications/sync` — Email sync + webhook

## Important Patterns
- **Sender email**: Use Graph `/me` to get connected account email, NOT `user.email` (which is the Supabase login)
- **HTML detection**: `/<[a-z][\s\S]*>/i.test(msg.bodyHtml)` — checks for actual HTML tags
- **bodyPreview storage**: Always strip HTML: `.replace(/<[^>]*>/g, "").substring(0, 255)`
- **Reply All**: Filters out `connectedEmails` (passed from parent) so dispatch@ doesn't appear in CC
- **HTML email rendering**: Sandboxed iframe with `sandbox=""` (no permissions) for XSS safety
- **Reply strategy**: Try Graph `/messages/{id}/reply` first, fall back to `/me/sendMail`
- **Sync dedup**: Batch `findMany({ where: { graphMessageId: { in: ids } } })` not N individual queries
- **Pagination**: Follow `@odata.nextLink`, up to 3 pages (300 emails max per sync)

## Schema
- `TicketComment` model added (id, ticketId, content, isInternal, createdBy, createdAt)
- ThreadCategory enum: `CUSTOMER`, `LEAD`, `BROKER`, `INTERNAL`, `SPAM`, `UNCATEGORIZED`

## Dependencies Added
- `@tiptap/react`, `@tiptap/starter-kit`, `@tiptap/extension-link`, `@tiptap/extension-placeholder`, `@tiptap/extension-underline`, `@tiptap/pm`
