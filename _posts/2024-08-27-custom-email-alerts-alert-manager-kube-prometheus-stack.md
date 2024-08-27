---
layout: post
title:  "How to Create a Custom Email Template for Alertmanager in Kube Prometheus Stack"
date:   2024-08-27
excerpt: "This guide walks you through the steps to create a concise, clear, and visually appealing alert email template, ensuring that important information is highlighted and unnecessary details are minimized"
tag:
- Kubernetes
- Prometheus
- Alertmanager
- Custom Email Template
- Monitoring
- Kube Prometheus Stack
- Alerting Configuration
- Alertmanager email templates
comments: true
---

# 📧 Custom Email Template for Alertmanager

When using the Kube Prometheus stack, the default email template in Alertmanager can be overwhelming, displaying all labels with crucial alert information buried at the bottom. To improve clarity and conciseness, we’ll configure a custom email template. This template will:

- Prioritize alert summaries.
- Display only important labels.
- Provide better content spacing for readability.

<figure>
    <a href="{{ site.url }}/assets/img/2024/08/alerts-sample-email.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/08/alerts-sample-email.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/08/alerts-sample-email.png">
            <img src="{{ site.url }}/assets/img/2024/08/alerts-sample-email" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}

## 🚀 Steps to Configure the Custom Template

### 1. Update the `tplConfig` in `values.yaml`

Ensure the `tplConfig` setting is set to `false` in the `alertmanager` spec. If set to `true`, it could interfere with email template rendering.

```yaml
alertmanager:
  tplConfig: false
```

### 2. Define the Custom Email Template

Under the `config` section in `alertmanager` spec, define your custom email template using the `templateFiles` variable. Below is an example using the Go template language.

{% raw %}
```yaml
alertmanager:
  templateFiles:
    custom_email.tmpl: |-
        {{ define "email.default.subject" }}{{ template "__subject" . }}{{ end }}
        {{ define "custom_email.html" }}
        <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
        <html xmlns="http://www.w3.org/1999/xhtml" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
        <head>
        <meta name="viewport" content="width=device-width">
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>{{ template "__subject" . }}</title>
        <style>
        @media only screen and (max-width: 640px) {
          body {
            padding: 0 !important;
          }
          h1,
        h2,
        h3,
        h4 {
            font-weight: 800 !important;
            margin: 20px 0 5px !important;
          }
          h1 {
            font-size: 22px !important;
          }
          h2 {
            font-size: 18px !important;
          }
          h3 {
            font-size: 16px !important;
          }
          .container {
            padding: 0 !important;
            width: 100% !important;
          }
          .content {
            padding: 0 !important;
          }
          .content-wrap {
            padding: 10px !important;
          }
          .invoice {
            width: 100% !important;
          }
        }
        </style>
        </head>
        <body itemscope itemtype="https://schema.org/EmailMessage" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; -webkit-font-smoothing: antialiased; -webkit-text-size-adjust: none; height: 100%; line-height: 1.6em; background-color: #f6f6f6; width: 100%;">
        <table class="body-wrap" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; background-color: #f6f6f6; width: 100%;" width="100%" bgcolor="#f6f6f6">
          <tr style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
            <td style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top;" valign="top"></td>
            <td class="container" width="600" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; display: block; max-width: 600px; margin: 0 auto; clear: both;" valign="top">
              <div class="content" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; max-width: 600px; margin: 0 auto; display: block; padding: 20px;">
                <table class="main" width="100%" cellpadding="0" cellspacing="0" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; background-color: #fff; border: 1px solid #e9e9e9; border-radius: 3px;" bgcolor="#fff">
                  <tr style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
                    {{ if gt (len .Alerts.Firing) 0 }}
                    <td class="alert alert-warning" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; vertical-align: top; font-size: 16px; color: #fff; font-weight: 500; padding: 20px; text-align: center; border-radius: 3px 3px 0 0; background-color: #E6522C;" valign="top" align="center" bgcolor="#E6522C">
                      {{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }} for {{ range .GroupLabels.SortedPairs }}
                        {{ .Name }}={{ .Value }}
                      {{ end }}
                    </td>
                    {{ else }}
                    <td class="alert alert-good" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; vertical-align: top; font-size: 16px; color: #fff; font-weight: 500; padding: 20px; text-align: center; border-radius: 3px 3px 0 0; background-color: #68B90F;" valign="top" align="center" bgcolor="#68B90F">
                      {{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }} for {{ range .GroupLabels.SortedPairs }}
                        {{ .Name }}={{ .Value }}
                      {{ end }}
                    </td>
                    {{ end }}
                  </tr>
                  <tr style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
                    <td class="content-wrap" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; padding: 30px;" valign="top">
                      <table width="100%" cellpadding="0" cellspacing="0" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
                        {{ if gt (len .Alerts.Firing) 0 }}
                        <tr style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">
                          <td class="content-block" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; padding: 0 0 20px;" valign="top">
                            <strong style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">[{{ .Alerts.Firing | len }}] Firing</strong>
                          </td>
                        </tr>
                        {{ end }}
                        {{ range .Alerts.Firing }}
                        <tr style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;">  
                          <td class="content-block" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; padding: 0 0 20px;" valign="top">  
                            {{ if gt (len .Annotations) 0 }}<strong style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><u>Alert Info</u></strong><br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Annotations.summary }}<strong>Summary:</strong> {{ .Annotations.summary }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Annotations.description }}<strong>Description:</strong> {{ .Annotations.description }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Annotations.runbook_url }}<strong>Runbook URL:</strong> <a href="{{ .Annotations.runbook_url }}" style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; color: #348eda; text-decoration: underline;">{{ .Annotations.runbook_url }}</a><br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            <strong style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><u>Labels</u></strong><br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>
                            {{ if .Labels.domain_name }}<strong>Domain Name:</strong> {{ .Labels.domain_name }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Labels.stage }}<strong>Stage:</strong> {{ .Labels.stage }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Labels.region }}<strong>Region:</strong> {{ .Labels.region }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}
                            {{ if .Labels.severity }}<strong>Severity:</strong> {{ .Labels.severity }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                            {{ if .Labels.value }}<strong>Value:</strong> {{ .Labels.value }}<br style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px;"><br>{{ end }}  
                          </td>  
                        </tr>
                        {{ end }}
                      </table>
                    </td>
                  </tr>
                </table></div>
            </td>
            <td style="margin: 0; font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top;" valign="top"></td>
          </tr>
        </table>
        </body>
        </html>
        {{ end }}

```
{% endraw %}

**Explanation**:

- The custom template `custom_email.tmpl` prioritizes the alert summary, description, and runbook URL, followed by selected labels (e.g., domain name, stage, region).
- Proper spacing is added for better readability.

When deploying the Helm chart, this file will be loaded into the `/etc/alertmanager/config` directory and used by Alertmanager.

**Note:** Ensure the template file ends with `.tmpl`, as this suffix is required for Alertmanager to pick it up.

### 3. Configure the Email Receiver

Finally, configure the email receiver to use the custom email template.

```yaml
- name: "email"
  email_configs:
  - to: "recipient@example.com"
    from: "alertmanager@example.com"
    html: '{{ template "custom_email.html" . }}'
```

With these steps, your Alertmanager will send concise and clear email alerts, making it easier to act on critical issues.

{% include donate.html %}
{% include advertisement.html %}