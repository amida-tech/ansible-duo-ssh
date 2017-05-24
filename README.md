# Multiuser SSH 2FA Via Duo
This repository contains an Ansible role used to set up [Duo](http://duo.com/)-based
2-factor authentication SSH logins. The role supports CentOS- and Debian- based systems, and has
been tested on latest (05/24) CentOS 7 and Ubuntu 16.04.

Duo is patched for multi-user support: any Duo user in the organization can authenticate as any user on the
system (provided they also have a matching SSH key etc).

If multiple humans use the same account on a system
(Alice and Bob both log in as `centos`) this is great -- they can each have their own Duo account and
hence own 2FA devices. If each human has their own user account, this probably isn't what you want and you
should use a [different](https://github.com/jlafon/ansible-duo-security) Ansible role.

Please read this README carefully to avoid creating security holes. In particular, note that you **must**
modify the default Duo _New User Policy_.

## Security Discussion
By default, Duo does not allow system-wide multi-user support (setting the `setuid` bit disables shared accounts for non-root-users, setting the `setuid` bit requires each user to have their own configuration file). This is for good reason: it allows every Duo user to retrieve the duo conf file, 
and hence fake authentication as other Duo users.

However, by making the assumption that all Duo users are equally trusted: i.e., **all Duo users in the
organization can perform the second factor of authentication for any account on the system**, then these
security concerns go away. In fact, the assumption is weaker: a group of users can be created in the Duo
dashboard, and the application restricted to this group. The trust then only need extend to users in this
group.

Additionally, the Duo **_New User Policy_ must be set to _Deny access to unenrolled users_**. Otherwise,
an attacker can simply enter a nonexistent Duo user name, receive a link to enroll into Duo and then use
their newly-created account to successfully complete 2FA.

Finally, OpenSSH initializes port forwarding and tunneling before the Duo 2FA challenge, so `PermitTunnel`
and `AllowTcpForwarding` are [disabled](https://duo.com/docs/loginduo) to avoid a potential attack via port
forwarding.


## Walkthrough
1. Log into, or create a new account on [duo.com](https://duo.com/).
2. Use the _Applications_ tab of the Dashboard to protect a new `UNIX Application`. Make note of the
integration key, secret key and API hostname.
3. Under the _Policy_ section of the application, create a new _Application Policy_. Set the _New User
Policy_ to _Deny access to unenrolled users._. 
4. Use the _Users_ tab of the dashboard to create a new Duo user. Add an email address, and follow the steps
in the welcome email to complete the enrollment process.
5. Install Ansible and run
```
ansible-playbook --extra-vars "ikey=IKEY skey=SKEY duo_host=DUO_HOST" -u USER -i "HOST," playbook.yml
```
where `USER@HOST` is the server to run on, and `IKEY`, `SKEY` and `DUO_HOST` are the [vars](#Variables)
noted in step 2.
6. Login to the server using the same key/etc as before. When prompted for a Duo user, enter the username
created in step 4. Follow the on-screen instructions to complete push/SMS/voice authentication.

## Usage
### Standalone
A barebones `playbook.yml` is included that can be used to run the role. Install Ansible and run
```
ansible-playbook --extra-vars "ikey=IKEY skey=SKEY duo_host=DUO_HOST" -u USER -i "HOST," playbook.yml
```
where `USER@HOST` is the machine to run on, and `IKEY`, `SKEY` and `DUO_HOST` are the vars described in
[Variables](#Variables).

### Integration into an Existing Ansible Playbook
To integrate the role into an existing ansible playbook:
1. Copy the `roles/duo` directoryfrom this repository into the existing ansible `roles` directory.
2. Add `duo` to the roles array in the existing playbook.
3. Add the duo [vars](#Variables) to the playbook or inventory. Alternatively, pass in the vars when
running the `ansible-playbook` command, as demonstrated in [Standalone](#Standalone).

## Variables
* `IKEY` - Duo application Integration Key (found in application dashboard)
* `SKEY` - Duo application Secret Key (found in application dashboard)
* `DUO_HOST` - Duo application host endpoint (found in application dashboard)
