
The ERD is live in LucidChart with **5 tabs**, one per entity group from the okf-bundle specs:

1. **Governance & Membership** — Society, Project, Member, Asset, Ownership, Employee, ManagementRole/EC, PC Membership, EmployeeAssetViewGrant
2. **Finance & Ledgers** — Charge, Payment, Receipt, JournalEntry, JournalPosting, LedgerAccount, plus the four books as derived views
3. **Vendors & Work Orders** — Vendor, WorkOrder, Invoice (with self-referential `resubmission_of`), InvoiceApproval
4. **Documents & Communication** — Document, DocumentCategory, LedgerDocumentLink, BulletinPost, DirectMessage, MessageRecipient
5. **Member Requests (Ticketing)** — Ticket, TicketComment, TicketAttachment, TicketRouting

Each entity is an editable table with PK/FK-annotated attributes and crow's-foot relationship lines. Grey "ref → other tab" stubs (Member, Employee, Project, WorkOrder) mark cross-group foreign keys so each page stays self-contained and readable.

Open it here: [https://lucid.app/lucidchart/5c9fe189-0314-434c-996c-2ef2441f9d91/edit](https://lucid.app/lucidchart/5c9fe189-0314-434c-996c-2ef2441f9d91/edit)