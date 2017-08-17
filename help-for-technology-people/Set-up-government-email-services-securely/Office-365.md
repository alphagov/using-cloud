## Set up government email services securely - Office 365
How to implement the guidance on [securing government email](https://www.gov.uk/guidance/securing-government-email) in [Microsoft Office 365](https://products.office.com/en-gb/government/office-365-web-services-for-government) to provide encryption, anti-spoofing, and to pass an assessment. Doing these things will add your domain to a whitelist of secure domains which organisations can use to filter email.

The [domain information tool](http://domaininformation.service.gov.uk/) is an alpha service that gives you a dashboard of the domains in your organisation, a way to check whether an email sent between two domains should be secure, and a whitelist of domains that are setup securely. Request access through the email assessment [ask a question form](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=130185).

For more detailed advice contact Microsoft or your IT service provider for help.
### Email service prerequisites
You will need:
* access to make administrative changes in Office 365
* a public DNS record you can make changes to for each email domain

You are not required to have additional perimeter email security services. The scanning and filtering service provided with Exchange Online should provide sufficient protection.
### Encryption
Follow these steps to encrypt email services:

1. use [Transport Layer Security (TLS)](https://www.gov.uk/government/publications/email-security-standards/transport-layer-security-tls) version 1.2 or later and [preferred cryptographic profiles](https://www.ncsc.gov.uk/guidance/tls-external-facing-services) for secure email transport between UK government departments.
 * this is enabled by default in Office 365

2. force a TLS connection from your sending domain.
 Do not rely on the recipient domain to do this. Create rules to use mandatory TLS when exchanging emails with government organisations, including *.gsi.gov.uk, *.gsx.gov.uk, *.gse.gov.uk, *.gcsx.gov.uk domains, domains included on the whitelist, and any other contacts that are known to support TLS such as commercial partners or suppliers.  To do this:
 * create a new connector in Office 365 Admin Centre | Exchange Admin Centre | mailflow | connectors
 * the connection should be from ‘Office 365’ to a ‘Partner Organisation’
 * give the connector a name and description
 * use the connector ‘Only when email messages are sent to these domains’ and add each domain in turn (eg *.gsi.gov.uk)
 * choose to always use TLS for this connection, and require certificates issued by a Trusted CA
 * add domains in groups to make it easy to administer - for example have a rule for the government domains listed above, and another to manage connections with other partner organisations
 * don't require the subject alternative name to match the domain name (this is desirable but hard to achieve in many case)
 
You need a CA signed certificate.

Ideally you will ingest the whitelist of domains and force TLS to those domains automatically, as the list will change over time.  You can do this using a powershell script or other mechanism to read the contents of the [public URL of the whitelist](https://domaininformation.service.gov.uk/white-list/export?separator=comma) and [create and maintain an email connector rule](https://technet.microsoft.com/en-gb/library/jj200761%28v=exchg.160%29.aspx?f=255&MSPPError=-2147217396).

3. do not create a connector to enforce TLS to *.gov.uk as a number of domains aren’t yet able to support it.

4. Microsoft use a [strong TLS cipher suite](https://technet.microsoft.com/en-gb/library/dn569286.aspx?f=255&MSPPError=-2147217396) and uses it’s own certificates to secure your connection

5. opportunistic TLS is enabled by default for domains not included in the mandatory TLS connectors created above. You can use self-signed certificates for opportunistic TLS.

6. show you have outbound TLS available and are using Domain Keys Identified Mail ([DKIM](https://www.gov.uk/government/publications/email-security-standards/domainkeys-identified-mail-dkim)) signing email by either creating an auto-reply or sending a scheduled email.

To create an auto-reply:

 * [create a shared mailbox](https://technet.microsoft.com/en-gb/library/jj150570(v=exchg.160).aspx) for each of your email domains (for example secureemailreply@yourdomain.gov.uk)
 * [tell us the address](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=130185) you are using for each domain
 * open Outlook using the profile of the shared mailbox. You cannot create the auto-reply when logged in as another user, even with full delegated access. You also need to authenticate using the shared mailbox credentials.  It'll let you log in with an admin user but you can't create the rule under that authentication. If you don't know the password for the shared mailbox you can reset it in the Exchange Admin Portal - it may take an hour or so after creation for the address to appear in the user list.
 * click on Rules - Create Rule - Advanced Options
 * apply these conditions:
 1. check for email from emailsecurity@domaininformation.service.gov.uk and sent specifically to your email address
 2. have the server reply using a specific message - put emailsecurity@domaininformation.service.gov.uk in the To: field with any subject and message body.
 3. don't apply any exceptions
 4. give the rule a name and turn it on


Do not use the Out of office or Automatic replies option as they only respond to the first message.

>After resetting the password you can complete the setup using the Outlook Web app. [Tell us the email address](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=130185), then use the email we send to create a rule and change the settings accordingly.

To send an email on a schedule use Windows Task Scheduler (or cron on Unix-based machines) to send an separator email every day from each domain you are responsible for. The email must have the correct sender information to make sure it is processed correctly - you can't spoof this email from another source.

### Anti-spoofing
To prevent email spoofing you must put technical and business policies in place to check inbound and outbound government email using Domain-based Message Authentication, Reporting and Conformance ([DMARC](https://www.gov.uk/government/publications/email-security-standards/domain-based-message-authentication-reporting-and-conformance-dmarc)).

1. [Implement DMARC](https://www.gov.uk/guidance/set-up-government-email-services-securely#create-and-iterate-dmarc-records) by:

 * publishing a DMARC record starting at ‘p=none’ rising to ‘p=quarantine’ during implementation
 * following the [configuration guidance](https://www.gov.uk/guidance/set-up-government-email-services-securely#create-and-iterate-dmarc-records)
 * enabling inbound DMARC validation - this happens automatically on your Office 365 domain, there are no configuration options for the administrator

2. [Implement Sender Policy Framework](https://www.gov.uk/guidance/set-up-government-email-services-securely#create-and-iterate-spf-records) ([SPF](https://www.gov.uk/government/publications/email-security-standards/sender-policy-framework-spf)) by publishing public DNS records for SPF, including all systems that send email, using a minimum soft fail (~all) qualifier

 [Create an SPF record for Office 365](https://support.office.com/en-gb/article/External-Domain-Name-System-records-for-Office-365-c0531a6f-9e25-4f2d-ad0e-a70bfef09ac0?ui=en-US&rs=en-US&ad=US&fromAR=1) using both IPv4 and IPv6 addresses.  A basic record for a domain that uses Office 365 for email and Sharepoint should look like this:
 <pre><code>v=spf1 include:spf.protection.outlook.com include:sharepointonline.com ~all</code></pre>
 You may need to add other domains and IP ranges to this record if your domain has other email sources. The Exchange Online Protect best practice guidance [also advises on SPF](https://technet.microsoft.com/en-gb/library/jj723164(v=exchg.150).aspx).

3. [Implement DKIM](https://www.gov.uk/guidance/set-up-government-email-services-securely#create-and-manage-dkim) by:

 * publishing DKIM selector and policy records 
 * signing outgoing email in accordance with the DKIM standard
 * disabling outbound email footers in your outbound email filtering service (if you have one)
 * create a [CNAME](https://en.wikipedia.org/wiki/CNAME_record) record in DNS for any domain aliases you have created (any email domain not using the default ‘onmicrosoft.com’ domain name)
 * enable DKIM in the Exchange Online admin panel

DKIM outbound is configured through the Exchange administration DKIM section.  As Office 365 is a multi-tenanted service Microsoft will generate the DKIM certificate on your behalf. This means your DKIM DNS records refer people back to a Microsoft URL rather than providing a key for comparison.  [Read Microsoft’s blog on the subject](http://blogs.msdn.com/b/tzink/archive/2015/10/08/manually-hooking-up-dkim-signing-in-office-365.aspx).

DKIM keys do not expire but should be rotated periodically.  Microsoft do this for you however so it is not necessary in Exchange Online. Similarly they manage the key size (which currently should be 1024-bit) so you don’t need to worry about that either.

If your outbound mail passes through a filtering service in addition to Exchange Online Protection you must ensure that service doesn’t alter the message headers (such as adding a disclaimer) as this will invalidate the DKIM signature. 

### Assessment
The domain information tool will check you have encryption and anti-spoofing configured.  You will also need to pass a cloud-based email service assessment to ensure your email service is configured and run in a secure way.

There is no requirement to route Office 365 email via the PSN to pass this assessment, even for gsi.gov.uk or gcsx.gov.uk email addresses. If you are moving a domain traditionally associated with the PSN use a simpler gov.uk domain name as you primary domain name and the legacy domain name as an alias. Pass the assessment with the primary domain name before moving the legacy alias across.

[Create a new ticket](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=134149) to request an email assessment and read the guidance in the form on the assessment process.

### Use the whitelist
Domains that have implemented the guidance and passed an assessment appear on a whitelist in the [domain information tool](http://domaininformation.service.gov.uk/).  You don’t have to use the whitelist but if you currently have rules to filter outbound email, for example limiting certain kinds of data to *.gsi.gov.uk domains, you should add the domain whitelist to these rules.

[Request access to the tool](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=130185) to access the whitelist.  It is available via a URL to help you include it in any automated processes (for example updating rules on your email service).  Use Powershell in your Office 365 environment to create and manage rules using the whitelist as the source.
