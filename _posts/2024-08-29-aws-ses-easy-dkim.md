---
layout: post
title:  "Configuring AWS SES to Send Emails to a Corporate Mail Server Using Easy DKIM"
date:   2024-08-29
excerpt: "Learn how to configure Amazon SES with Easy DKIM for secure corporate email delivery."
tag:
- AWS SES
- DKIM
- Email Authentication
- Corporate Email
- SES Configuration
- Easy DKIM
- DNS Configuration
comments: true
---

# DKIM

**DomainKeys Identified Mail (DKIM)** is an email authentication method that ensures an email claiming to come from a specific domain was indeed authorized by the domain's owner. 

DKIM uses public-key cryptography, signing an email with a private key. 

Recipient servers then use a public key published in the domain's DNS to verify that the email has not been tampered with during transit.

## Easy DKIM in SES

Amazon Simple Email Service (SES) offers **Easy DKIM**, where SES automatically generates a public-private key pair and adds a DKIM signature to every email sent from your verified identity. 

This simplifies the setup process, particularly when integrating with corporate mail servers.

### How Easy DKIM Works with SES

When you set up **Easy DKIM** for a domain identity in Amazon SES, SES automatically generates a public-private key pair and adds a 2048-bit DKIM signature to every message you send from that identity. This ensures the authenticity of your emails and improves deliverability.

### Setting Up Easy DKIM for Your Domain

1. **Sign in to the AWS Management Console** and open the [Amazon SES console](https://console.aws.amazon.com/ses/).

2. **Navigate to Verified Identities**: In the navigation pane, under "Configuration," choose **Verified identities**.

3. **Select Your Domain**: From the list of identities, choose the domain you want to configure.

4. **Enable Easy DKIM**:
   - Under the **Authentication** tab, in the **DomainKeys Identified Mail (DKIM)** container, click **Edit**.
   - In the **Advanced DKIM settings** container, choose the **Easy DKIM** option.
   - Set the **DKIM signing key length** to **RSA_2048_BIT** for enhanced security.
   - Ensure the **DKIM signatures** field is checked as **Enabled**.
   - Click **Save changes**.


{% include donate.html %}
{% include advertisement.html %}

### Publishing DKIM Records to Your Corporate DNS

While Amazon SES can automatically publish DKIM records to Route 53 DNS, if you are using a corporate DNS, you'll need to publish these records manually. Here’s how:

1. **Get the DKIM Records**: After enabling DKIM in SES, you’ll be provided with DKIM CNAME records that need to be added to your DNS.

2. **Publish to Corporate NameServers**: Add the provided CNAME records to your corporate DNS.

3. **Verify the Records**: To ensure that the records are published correctly, you can use the `dig` command. Here’s how:

   ```bash
   dig NS <subdomain>
   ```
   - This command will return the nameservers for your subdomain, ensuring they are correct.

   ```bash
   dig CNAME <DKIM_CNAME_NAME> @<NAMESERVER> +short
   ```
   - **Breakdown**:
     - **`CNAME <DKIM_CNAME_NAME>`**: Queries the CNAME record associated with your DKIM entry.
     - **`@<NAMESERVER>`**: Specifies the DNS server to query. Replace `<NAMESERVER>` with the actual DNS server address (e.g., `ns1.example.com`).
     - **`+short`**: This option simplifies the output, displaying only the essential information, i.e., the CNAME value.

   - **Example**: 
     ```bash
     dig CNAME abc123._domainkey.example.com @ns1.example.com +short
     ```
     This command would return just the CNAME value, confirming that the DKIM record is correctly published.

### Verifying DKIM Setup in SES

Amazon SES will periodically check if the DKIM records have been successfully published in your DNS servers. Once verified, the DKIM configuration status will show as **Successful**.

If the records aren’t verified:
- You can manually trigger the verification by deleting and re-creating the records. The same record values will be generated, so there’s no risk in this step.

### Visualizing the DKIM Process

Here’s a visual representation of the DKIM process to help you understand how emails are authenticated:

- The **Sending Mail Server** uses a private DKIM key to sign outgoing emails.
- The **Receiving Mail Server** extracts the DKIM signature from the email headers and queries the DNS for the public DKIM key.
- The DNS returns the public DKIM record, and the mail server uses it to verify that the email has not been altered.
- If the verification is successful, the email passes DKIM authentication.


{% include donate.html %}
{% include advertisement.html %}

### Sending Emails via SES

Once DKIM is successfully verified, you can start sending emails using SES. There are two main approaches:

1. **SES API:**
   - Simple and efficient for integrating email sending capabilities directly into your application.

2. **SMTP:**
   - Useful when integrating with existing email clients or servers that rely on SMTP.

#### Example CLI Command for Sending Email

Use the following CLI command to send a test email and verify that it's delivered to the recipient's mail server:

```bash
aws sesv2 send-email --from-email-address test@appmail.example.com --destination "ToAddresses=admin@example.com" --content file://message_simple.json --region us-east-1
```

The `message_simple.json` file should contain:

```json
{
  "Simple": {
    "Subject": {
      "Data": "Test Email",
      "Charset": "UTF-8"
    },
    "Body": {
      "Text": {
        "Data": "This is a test email.",
        "Charset": "UTF-8"
      }
    }
  }
}
```

{% include donate.html %}
{% include advertisement.html %}