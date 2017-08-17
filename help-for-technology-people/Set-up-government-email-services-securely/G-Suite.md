## Set up government email services securely - G Suite

How to implement the guidance on [securing government email](https://www.gov.uk/guidance/securing-government-email) in G Suite (formerly Google Apps) to provide encryption, anti-spoofing, and to pass an assessment. Doing these things will add your domain to a whitelist of secure domains which organisations can use to filter email.

The [domain information tool](http://domaininformation.service.gov.uk/) is an alpha service that gives you a dashboard of the domains in your organisation, a way to check whether an email sent between two domains should be secure, and a whitelist of domains that are setup securely.  Request access through the email assessment [ask a question form](https://emailassurance.zendesk.com/hc/en-us/requests/new?ticket_form_id=130185).

For more detailed advice contact Google or your IT service provider for help.

### Email service prerequisites

You will need:

* access to make administrative changes in G Suite
* a public DNS record you can make changes to for each email domain

You are not required to have additional perimeter email security services.  The scanning and filtering service provided with Gmail should provide sufficient protection.

### Transport Layer Security (TLS)

[Create compliance rules](https://support.google.com/a/answer/2520500?hl=en) to enforce TLS from your email service to:

* \*.gsi.gov.uk, \*.gsx.gov.uk, \*.gse.gov.uk, \*.gcsx.gov.uk, \*.secure.nhs.net
* the [whitelist of domains](https://domaininformation.service.gov.uk/white-list) in the domain information tool
* any other contacts that are known to support TLS such as commercial partners or suppliers



Don't create a rule to enforce TLS to *.gov.uk as a number of domains aren't yet able to support it.

Don't rely on the recipient to force the TLS connection.

G Suite [manages TLS certificates](https://support.google.com/a/answer/6180220?hl=en&ref_topic=2683824) on your behalf.  Opportunistic TLS is also [on by default](https://support.google.com/a/answer/60762?hl=en).

This is an example compliance rule for TLS in G Suite: 

![screen shot of compliance setting](https://github.com/cheyrou23/using-cloud/blob/master/images/G%20Suite%20Secure%20Transport%20Compliance%20rule.png)

#### Create an auto-reply

To ensure your email service is capable of sending and receiving email using TLS you must create an email address, eg emailsecurity@yourdomain.gov.uk, for each of your email domains. Create a rule or filter for this address so that if it receives an email from: emailsecurity@domaininformation.service.gov.uk it will reply automatically. The contents of the reply do not matter but it is important that the reply comes from the domain to which the message was sent.

This will be used to check outbound TLS and DKIM signing from each domain.

To do this in G Suite [create a Google Group](https://support.google.com/groups/answer/2464926?hl=en).  Set the group so you can post from outside the organisation.  Once created go to Manage > Settings > Email options > and check the auto-reply box for Enable auto-reply message for non-members outside the organization.  If required add some text to the box below.

Note that this will currently only generate an auto-reply for the primary domain.

### Domain-based Message Authentication, Reporting and Conformance (DMARC)

Follow the [implementation guidance](https://www.gov.uk/guidance/set-up-government-email-services-securely) on setting up DMARC in a cloud-based email service.

You can check the DMARC record in your DNS record using the domain information tool or a [DMARC lookup tool](https://dmarcian.com/dmarc-inspector/).

Check your DNS records to ensure you know what is there currently.  Records may have been created previously that do what you want, or may conflict with records you are about to create.

Create DMARC records for outbound email.  DMARC records are TXT records on your DNS.  A basic record for G Suite looks like this:

<pre><code>v=DMARC1; p=none; rua=mailto:email_address@domain.gov.uk</code></pre>

This means this is a DMARC record (v=DMARC1) , take no action on my email (p=none), and send reports to the email address shown.

To make it visible to others you need to name the record _dmarc.<your_domain>.  So when creating the TXT record it should look like this:

![DMARC DNS TXT Record](https://github.com/cheyrou23/using-cloud/blob/master/images/Setting%20up%20government%20email%20services%20securely%20in%20G%20Suite.png)

Use an email address in the same domain as the one you are protecting.  If you use a different domain you need to include a TXT record in the DNS record of the receiving domain as well.

Name the record:

<pre><code>sending_domain.gov.uk._report._dmarc.recipient_domain.gov.uk</code></pre>

with record content:

<pre><code>v=DMARC1</code></pre>

See the [RFC](https://tools.ietf.org/html/rfc7489#section-7.1) for more detail on how DMARC records are structured.

Inbound DMARC validation happens automatically on your G Suite domain - there are no configuration options for the administrator.

After a day or so you will start receiving email reports from [major email recipient domains](http://dmarc.io/sources/).  These are mainly webmail providers.  Other government organisations or businesses do not provide these reports, although organisations using cloud-based email services will have these sent on their behalf by their provider.  These reports are in XML and can be difficult to read but there are services available to help you understand them.

This is the start of the DMARC implementation process.  Use the information the DMARC  to ensure you are aware of and control all outbound email from your domains.  You may need to make changes to the configuration of your email services until you are satisfied that genuine email is being correctly identified.

There are more resources to help create a DMARC record:

* [DMARC record assistant](http://kitterman.com/dmarc/assistant.html)
* [DMARC tag registry](https://dmarc.org//draft-dmarc-base-00-01.html#iana_dmarc_tags)
* [DMARC record format](https://dmarc.org//draft-dmarc-base-00-01.html#dmarc_format)

Follow DMARC guidance to audit email servers and ramp up restriction until you reach p=reject.  This should can take three to six months depending on the complexity of your environment.  Implementing the rest of this blueprint is part of that process.

If your users make use of mailing lists message delivery could be affected by the introduction of DMARC.  In this case mailing list providers should avoid sending forensic DMARC reports (ruf=) and your organisation should consider [other ways](https://dmarc.org/wiki/FAQ#I_operate_a_mailing_list_and_I_want_to_interoperate_with_DMARC.2C_what_should_I_do.3F) to mitigate this issue.

### DomainKeys Identified Mail (DKIM)

Check if you have existing DKIM records.  You need to know the [DKIM selector](http://dkim.org/info/dkim-faq.html) to find the record using a [DKIM lookup tool](http://dkimcore.org/tools/keycheck.html).  In Google Apps this defaults to 'google' or check the headers in an email from your domain to find the selector.

You should [configure DKIM](https://support.google.com/a/answer/174124?hl=en) first to ensure delivery of calendar notifications from your Google Apps domain.  This will sign all outbound messages using a key generated through the Google Apps interface.

DKIM keys do not expire but should be rotated periodically.  Create a new key with a new selector and follow the same steps as above.  Keep the old DNS record live for a few days after making changes to give DNS time to update.

Google will automatically generate a DKIM key for you which you can then use to create a TXT record in the DNS record for the email domain.

The TXT record should look like this:

![DKIM DNS TXT Record](https://github.com/cheyrou23/using-cloud/blob/master/images/DKIM%20DNS%20Record%20example.png)

If your outbound mail passes through a filtering service in addition to Google Apps you must ensure that service doesn't alter the message headers (such as adding a disclaimer) as this will invalidate the DKIM signature.  There is more guidance here:

[The importance of rotating your DKIM keys](https://www.sparkpost.com/blog/the-importance-of-rotating-your-dkim-keys/)
[Explaining DKIM to your grandmother](https://www.sparkpost.com/blog/explaining-dkim-to-your-grandmother/)
[Generating DKIM keys](http://domainkeys.sourceforge.net/keygen.html)

Emails signed with DKIM have a better chance of delivery but initially signing mail from one source and not another will not negatively affect mail delivery from any of your sources until you publish a DMARC quarantine or reject record.  You can further control this by publishing an ADSP record which tells recipient domains whether to expect your email to be DKIM signed.

### Sender Policy Framework (SPF)

[Create an SPF record](https://support.google.com/a/answer/178723?hl=en).  You can use both IPv4 and IPv6 addresses.  A basic record for a domain that only uses Google Apps for email should look like this:

<pre><code>v=spf1 include:_spf.google.com ~all</code></pre>

The 'all' indicates the action recipients should take on validating the email.
>?all means take no action
~all means mark but don't reject email that doesn't validate,
-all means reject invalid email.

This setting is overwritten by DMARC validation in most cases.

It is likely you will need to add other domains and IP ranges to this basic record to ensure continued email delivery for your domain.  This is an important and ongoing process for maintaining an accurate SPF record and ensuring delivery of your email.  If you omit a service from your SPF record recipient domains may reject email sent from that service.

A DNS record for SPF including another provider (in this case MailChimp) looks like this:

![SPF DNS TXT Record](https://github.com/cheyrou23/using-cloud/blob/master/images/SPF%20DNS%20Record%20example.png)

### Other email sending services
Refer to the [generic implementation guide](https://www.gov.uk/guidance/set-up-government-email-services-securely#configure-other-email-sending-services) for help with other email sending services.

### Making DNS changes
Refer to the [generic implementation guide](https://www.gov.uk/guidance/set-up-government-email-services-securely#make-dns-changes) for help making DNS changes.

### Communicating these changes to your organisation
Refer to the [generic implementation guide](https://www.gov.uk/guidance/set-up-government-email-services-securely) for help communicating changes.

### Assessment
Refer to the [generic implementation guide](https://www.gov.uk/guidance/set-up-government-email-services-securely) for information on assurance
