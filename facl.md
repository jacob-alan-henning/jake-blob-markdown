# Postmortem: An Ansible module lied to me

## What

Blog was down for about an hour. No one could read anything.

## Why

A few months ago I switched the blog container from running as root to a non-root user. One less workload on a small vps, better security, it made me feel warm and fuzzy.

That meant the container now reads TLS certificates directly from the host filesystem as a non-root user. The certificates live in `/etc/letsencrypt`, owned by root. So the container needs permissions to read the private key. ACL's are needed beyond normal unix perms because certbot will override them on renewal but leave ACL's.

I added Ansible ACL tasks to my playbook. Ran it. Green. Blog worked.

For about a month everything was fine. The module kept reporting ok every run. I assumed it was doing its job. It wasn't. The existing cert files already had ACLs from a previous manual setup. Certbot just hadn't renewed yet.

Time bomb.

### Why staging didn't catch it

I have a dynamically created staging environment that runs the exact same playbook. Only the parameters are different. It didn't catch this.

The module bug only triggers when files already exist in the directory. Staging spins up fresh every time with new certs in an empty directory. No existing files means no `*,*` in the setfacl output means the module actually works. The bug only appears when you're applying default ACLs recursively to a directory that already contains files. Which is exactly what happens in prod after certbot renews.

## When

Saturday night. Certbot renewed the certificate, created fresh files with no ACLs, my maintenance script restarted the service, and the service tried to read the new private key. Couldn't.

I caught it within an hour. I check my cost dashboard weekly around when the maintenance script runs and noticed the site was timing out. Could have been way worse.

After triage I believe the error was clear; blog process doesn't have rights to read the cert files. Thats odd; I'll rerun the workflow it will kick off the playbook. That should set the perms correct.

```
TASK [Ensure certreaders can read existing cert files] ********
ok: [server] => (item=live)
ok: [server] => (item=archive)

TASK [Ensure future cert files inherit ACLs] ******************
ok: [server] => (item=live)
ok: [server] => (item=archive)
```

Green checkmarks. Everything fine apparently. Blog still down.

The module wasn't actually setting anything. It checked if changes were needed, got a false negative, and reported success. Every run. For months.

I ssh'd in and manually added the acls to the renewed certs. Bleeding stopped; come back later and find the cause because something is really wrong.

## Root cause

### How ACLs work

POSIX Access Control Lists let you grant file permissions to specific users or groups without changing ownership. You have a file owned by root, you want a non-root service to read it:

```bash
setfacl -m g:mygroup:r /path/to/file
```

Default ACLs go on directories and get inherited by new files:

```bash
setfacl -d -m g:mygroup:rX /path/to/directory
```

That `-d` flag means "default." Any file created in that directory gets the ACL applied. This is how you handle certificate renewals or anything that creates new files needing specific permissions.

### How the module broke

Ansible has `ansible.posix.acl` for this. Declarative YAML:

```yaml
- name: Ensure mygroup can read future files
  ansible.posix.acl:
    path: /path/to/directory
    entity: mygroup
    etype: group
    permissions: rX
    default: yes
    recursive: yes
    state: present
```

The module runs `setfacl --test` to check whether changes are needed:

```
setfacl --test --recursive --modify d:g:mygroup:rX .
.: *,d:g::r-x,d:g:mygroup:r-x,d:m::r-x,d:o::r-x
./somefile.pem: *,*
```

First line shows the directory would get the default ACL. Second line shows `*,*` for the file because default ACLs don't apply to files, only directories.

The module checks for `*,*` in the output. If it sees it, it concludes nothing needs to change.

But when you recurse, `*,*` appears for every file. So if the directory contains any files the module decides everything is already correct and reports `ok`.

```python
if line.endswith('*,*') and not use_nfsv4_acls:
```

Silent failure for `recursive: yes` with `default: yes` on a directory containing files.

GitHub issue #592, filed November 2024: *"Default ACL are not set recursively if file is present in subfolder."* Eighteen reactions. PR #638. Still open as of February 2026.

## Resolution

Shell it out

```yaml
- name: Set ACLs for certreaders
  ansible.builtin.shell: |
    setfacl -m g:certreaders:rX /etc/letsencrypt/live
    setfacl -m g:certreaders:rX /etc/letsencrypt/archive
    setfacl -R -m g:certreaders:rX /etc/letsencrypt/live/{{ domain }}
    setfacl -R -m g:certreaders:rX /etc/letsencrypt/archive/{{ domain }}
    setfacl -d -m g:certreaders:rX /etc/letsencrypt/live/{{ domain }}
    setfacl -d -m g:certreaders:rX /etc/letsencrypt/archive/{{ domain }}
  changed_when: true
```

A `getfacl` after the playbook ran would have shown me the problem. I didn't check because the output was green. The maintenance script also sets ACLs directly after cert renewal now as a backstop.

We're told shelling out is lazy. Ugly. Real Ansible means modules and idempotency.

When a module wrapping a decades old Unix tool introduces a bug the underlying tool doesn't have and then incorrectly reports success during a common use case; why even bother? 

How many times will I need to learn the lesson. Leave Ansible playbooks as the hacky sloppy scripts they are.

