# MMLI Connect — Backend Setup (Supabase)

The portal in `membership.html` is a static frontend (fine for GitHub Pages).
It is built to talk directly to a **Supabase** project for storage and the
database. Until you complete the steps below, the portal runs in **demo
mode**: the full wizard works, but nothing is saved anywhere.

```
MMLI Website (GitHub Pages)
        ↓
MMLI Connect Portal (membership.html + assets/mmli-connect.js)
        ↓
Supabase (Postgres + Storage, secured with Row Level Security)
        ↓
MMLI Admin Dashboard (build separately, using the service-role key server-side only)
```

## 1. Create the project

1. Create a free Supabase project at supabase.com.
2. In **Project Settings → API**, copy the **Project URL** and the
   **anon / public key**.
3. Open `assets/mmli-connect.js` and fill in the top of the file:

   ```js
   const SUPABASE_CONFIG = {
     url: "https://YOUR-PROJECT.supabase.co",
     anonKey: "YOUR-PUBLIC-ANON-KEY"
   };
   ```

   **Never** put the `service_role` key in `membership.html` or any file
   published to GitHub Pages — the anon key is the only key that is safe
   to ship in frontend code, because Row Level Security (below) controls
   what it's allowed to do.

## 2. Database schema

Run this in the Supabase SQL editor:

```sql
create table applications (
  id uuid primary key default gen_random_uuid(),
  application_reference text unique not null,
  application_type text not null check (application_type in ('membership', 'partnership')),
  category text not null,
  status text not null default 'Submitted'
    check (status in ('Submitted', 'Under Review', 'Additional Information Required',
                       'Approved', 'Declined', 'Withdrawn')),
  applicant_name text,
  email text,
  phone text,
  organization_name text,
  profile_data jsonb not null default '{}',
  engagement_data jsonb not null default '{}',
  photo_path text,
  logo_path text,
  document_paths jsonb not null default '[]',
  consent jsonb not null default '{}',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index applications_type_category_idx on applications (application_type, category);
create index applications_status_idx on applications (status);
create index applications_reference_idx on applications (application_reference);

-- keep updated_at current
create or replace function set_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger applications_set_updated_at
before update on applications
for each row execute function set_updated_at();
```

`profile_data` and `engagement_data` hold the category-specific fields as
JSON, so you don't need a new column for every question — the frontend
already sends a clean object shaped by whichever category was selected.

## 3. Storage buckets

Create three **private** buckets (Storage → New bucket → uncheck "Public
bucket"):

| Bucket             | Contents                                   |
|---------------------|---------------------------------------------|
| `passport-photos`  | Individual applicants' passport photographs |
| `org-logos`        | Institution / organization logos            |
| `documents`        | CVs, certificates, proposals, letters       |

Because the buckets are private, files are never publicly reachable by
URL — only the path is stored on the `applications` row, and your admin
dashboard fetches files with a signed URL using the service-role key on
a trusted server (never in the browser).

## 4. Row Level Security (RLS)

Enable RLS on `applications` and on each storage bucket, then add:

```sql
alter table applications enable row level security;

-- Public/anon users may INSERT a new application, but can never read,
-- list, update or delete any application (including their own) once
-- submitted. All review happens through the admin dashboard using the
-- service-role key on a trusted server.
create policy "public can submit applications"
on applications for insert
to anon
with check (true);
```

```sql
-- Storage: anon users may upload into these buckets but cannot list,
-- read, or overwrite existing files.
create policy "anon can upload passport photos"
on storage.objects for insert
to anon
with check (bucket_id = 'passport-photos');

create policy "anon can upload org logos"
on storage.objects for insert
to anon
with check (bucket_id = 'org-logos');

create policy "anon can upload documents"
on storage.objects for insert
to anon
with check (bucket_id = 'documents');
```

Do **not** add a `select`, `update`, or `delete` policy for the `anon`
role on `applications` or on these buckets — that is what keeps
applicant data (including photos) private from the public. Your admin
dashboard should authenticate separately (e.g. Supabase Auth with a
staff account, or a small server-side app using the `service_role` key)
and read through that trusted path instead.

## 5. What the frontend sends

On submit, `mmli-connect.js`:

1. Uploads the photo (if any) to `passport-photos/photos/<reference>-<filename>`.
2. Uploads the logo (if any) to `org-logos/logos/<reference>-<filename>`.
3. Uploads any supporting documents to `documents/documents/<reference>-<filename>`.
4. Inserts one row into `applications` with the reference, category,
   profile/engagement JSON, and the file **paths** (not public URLs).

If Supabase isn't configured, this step is skipped and a console message
explains that the portal is in demo mode — the success screen still
appears so the flow can be tested end-to-end.

## 6. Testing checklist

**Desktop**
- [ ] Membership → each of the 12 categories renders the correct fields
- [ ] Partnership → each of the 21 categories renders the correct fields
- [ ] Back/Next preserves previously entered data
- [ ] Required-field validation blocks progress and highlights the field
- [ ] Passport photo / logo upload: preview, replace, remove, oversized file, wrong file type
- [ ] Supporting documents: add multiple, remove one, oversized file rejected
- [ ] Review screen "Edit" buttons return to the correct step with data intact
- [ ] Both consent checkboxes are required before submission
- [ ] Success screen shows a unique `MMLI-YYYY-XXXXXX` reference
- [ ] "Start a New Application" fully resets the wizard

**Mobile (Android + iOS)**
- [ ] No horizontal scrolling on any step
- [ ] Buttons and tap targets are comfortably sized
- [ ] Camera/gallery picker opens for the photo upload
- [ ] Step tracker collapses to the condensed "Step X of 6" bar

**Accessibility**
- [ ] Every field has a visible label
- [ ] Tab order is logical; focus is visible on every interactive element
- [ ] Error messages are announced (region uses `aria-live="polite"`)
- [ ] `prefers-reduced-motion` disables the hero and step animations

**Backend (after Supabase is configured)**
- [ ] A test membership submission creates a row with the correct JSON shape
- [ ] A test partnership submission creates a row with the correct JSON shape
- [ ] Uploaded files appear in the correct private bucket/path
- [ ] Anonymous key cannot read, list, or download other applicants' files
- [ ] Anonymous key cannot `select`/`update`/`delete` rows in `applications`

## 7. Suggested admin dashboard (not included here)

Build this as a **separate, authenticated** app (it should not live in
the public GitHub Pages repo). It should:

- Authenticate staff (Supabase Auth).
- Use the `service_role` key **only** on a server/edge function, never
  in a browser bundle.
- List/filter `applications` by `application_type`, `category`, and
  `status`; update `status` as applications move through review.
- Generate short-lived signed URLs to view a photo, logo, or document.
