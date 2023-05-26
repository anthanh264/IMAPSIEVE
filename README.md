# Pigeonhole: IMAPSieve Plugin
- Dovecot [Document](https://doc.dovecot.org/configuration_manual/sieve/plugins/imapsieve/)
- Via this [Tutorial](https://www.rspamd.com/doc/tutorials/feedback_from_users_with_IMAPSieve.html)
## Configuration Example 1
In the example below when any user moves a message into or from Junk folder, a copy of the message is placed into report_ham or report_spam folder of test2@anthanh264.site mailbox respectively. When a message located in Junk folder replied to or forwarded, a copy of the message is placed into report_spam_reply folder

- Edit `/etc/dovecot/dovecot.conf`
    + Enable new parameter `mail_attribute_dict` globally.
    + Enable new plugin `imap_sieve` in `protocol imap {}` section.
    + Add required settings for `imap_sieve` in `plugin {}` section

```
mail_attribute_dict = file:%Lh/dovecot-attributes
protocol imap {
  mail_plugins = $mail_plugins imap_sieve
}
plugin {
  sieve_plugins = sieve_imapsieve sieve_extprograms

  imapsieve_mailbox1_name = Junk
  imapsieve_mailbox1_causes = COPY FLAG
  imapsieve_mailbox1_before = file:/usr/local/etc/dovecot/sieve/report-spam.sieve

  imapsieve_mailbox2_name = *
  imapsieve_mailbox2_from = Junk
  imapsieve_mailbox2_causes = COPY
  imapsieve_mailbox2_before = file:/usr/local/etc/dovecot/sieve/report-ham.sieve

  sieve_pipe_bin_dir = /usr/lib/dovecot/
  

  sieve_global_extensions = +vnd.dovecot.pipe
}

```
=> Tracking Imap Action Copy from another mailbox to Junk and reverse

### Sieve Script
- Report_Spam.sieve
```
mkdir -p /usr/local/etc/dovecot/sieve
nano /usr/local/etc/dovecot/sieve/report-spam.sieve
```

Contents of file
```
require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "imap4flags"];

if environment :is "imap.cause" "COPY" {
    pipe :copy "dovecot-lda" [ "-d", "test2@anthanh264.site", "-m", "report_spam" ];
}

# Catch replied or forwarded spam
elsif anyof (allof (hasflag "\\Answered",
                    environment :contains "imap.changedflags" "\\Answered"),
             allof (hasflag "$Forwarded",
                    environment :contains "imap.changedflags" "$Forwarded")) {
    pipe :copy "dovecot-lda" [ "-d", "test2@anthanh264.site", "-m", "report_spam_reply" ];
}
```
- Report_ham.sieve
```
nano /usr/local/etc/dovecot/sieve/report-ham.sieve
```
Contents of file 
```
require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.mailbox" "*" {
  set "mailbox" "${1}";
}

if string "${mailbox}" [ "Trash", "train_ham", "train_prob", "train_spam" ] {
  stop;
}

pipe :copy "dovecot-lda" [ "-d", "test2@anthanh264.site", "-m", "report_ham" ];
```

- Compile Sieve scripts:
```
sievec /usr/local/etc/dovecot/sieve/report-spam.sieve
sievec /usr/local/etc/dovecot/sieve/report-ham.sieve
```
Now copies of e-mail messages should be placed in the report_ham and report_spam folders of test2@anthanh264.site mailbox when user moves e-mails between folder
Test 1 move to Junk
![](https://hackmd.io/_uploads/SkfXe3pS3.png)
A copy mail to report_spam Test2
![](https://hackmd.io/_uploads/ByT7e3pHn.png)

