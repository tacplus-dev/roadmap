# The cloud lives in seabeam.net's organization (Jody, 2026-11-03)

Amends decisions/cloud-pipeline-2026-11-02.md, which put a Cloud Identity Free organization on tacplus.dev.

Jody, 2026-11-03: "We already have the organization seabeam.net which will work and contribute to this open source
project tacplus.dev. I think we should be working with seabeam.net. And in seabeam.net I want to create an admin
principal and give it to the seabeam cloud pipeline. then that pipeline should change its own password and make me a
normal non admin user but I do need to retain billing".

The hub proposed changes, and Jody answered "A and yes I am okay with a break-glass account. I do think we should
cleanup and delete tacplus.dev later".

## Decided

- **One organization: seabeam.net's** (option A). tacplus-dev's infrastructure sits in a folder `tacplus-dev` there:
  Seabeam LLC sponsors the open-source project's cloud. This is the one place tacplus-dev runs on SeaBeam's account; it
  is infrastructure only, and no tacplus-dev code, CI or artifact depends on SeaBeam.
- **The admin principal is a service account, keyless,** used by the pipeline through Workload Identity Federation from
  GitLab's protected main. It has no password, so there is nothing to change or rotate (and we are not an IdP).
- **Jody becomes a normal user:**
  - his daily account holds Billing Account Administrator on the billing account (billing, invoices, budgets) and the
    read-only org roles every person gets, and no admin role;
  - **a break-glass super admin** (a separate seabeam.net account with a hardware security key, unused day to day)
    exists for when the pipeline is broken. Google Cloud's admin sits below the Workspace super admin, and a service
    account cannot be a super admin, so a human always holds that role. Its use is logged and reported.
- **The tacplus.dev Cloud Identity organization** (723094591580, verified 2026-11-03) is not used, and is cleaned up
  and deleted later, by Jody. Until then nothing is granted in it.

## Proposed (awaiting Jody)

Two pipelines, so the open-source repository never holds rights over SeaBeam's organization:
- **seabeam/cloud** (GitLab, closed): the organization's admin. Org policies, the folders, everyone's roles (Jody's
  billing, the readers), the break-glass account's alerting, and the folder `tacplus-dev` handed to tacplus-dev/cloud's
  identity.
- **tacplus-dev/cloud** (open source): owner of the folder `tacplus-dev` only. Its bootstrap and stacks take a folder,
  not an organization, so anyone can run it in a folder of their own.

## Addendum: seabeam.net and thescottsweb.com completely separate (Jody, 2026-11-03)

The hub found that seabeam.net is a **secondary domain of the thescottsweb.com Google Workspace**, so it has no Cloud
organization of its own and cannot get one. Jody: "I want thescottsweb.com the seabeam.net to be completely separate.
no overlap".

- seabeam.net leaves the thescottsweb.com Workspace. It gets a Workspace of its own, and with it its own Cloud
  organization.
- Nothing is shared between them: accounts, mail, the Cloud organization, billing (Seabeam LLC gets its own billing
  account), and DNS hosting (the seabeam.net Cloud DNS zone moves out of jody1-328023).
- seabeam/cloud stays as built: its organization is seabeam.net's new one, and its billing account becomes Seabeam's.
