---
title: Performance alerts
description: Setting up and using proactive alerts
---

Engine alerts allow you to set thresholds on the performance data provided by
Engine and send notifications to Slack or PagerDuty when problems arise. They
are often useful for detecting anomalies, especially around releases or during
issues with an upstream provider. Alerts can be configured to monitor the
following metrics for your entire GraphQL service or individual operations:

- **Request rate** — requests per minute
- **Request duration** — p50/p95/p99 service time
- **Error rate** — errors per minute
- **Error percentage** — the number of requests with errors divided by total
  requests

These triggers you select are evaluated on a rolling five minute window. For
example, you configure an alert to trigger when an operation’s error rate
exceeds 5%. In production, if 6 out of 100 requests result in an error during
last five minute, then the alert will trigger with an error rate of 6%. Once
you have put out the fire and the five minute error rate falls back below 5%,
the alert will resolve. Here is an example of a Slack alert:

<div style="text-align:center">
![Slack Alert](../img/alerts/slack-alert.png)
</div>
<h2 id="setup">How to set up alerts</h2>

Navigate to the _Alerts_ tab in the _Metrics_ area of your service.

<div style="text-align:center">
![Welcome Alerts Page](../img/alerts/welcome-alerts-page.png)
</div>
<br></br>
Choose _Slack_ or _PagerDuty_, then follow the directions to create a Slack
Incoming Webhook or a PagerDuty integration key.

<div style="text-align:center">
![Configure Channel](../img/alerts/configure-channel.png)
</div>
<br></br>
Two sample alerts are created automatically, but must be enabled to take effect:

* A performance alert which triggers after consistently slow operation response times across all operations.
* An error alert which triggers during a period of high error rates (more than 5%) across all operations.

These sample alerts can be enabled and disabled by toggling the checkbox in the _Enabled_ column.

<div style="text-align:center">
![Default Alerts](../img/alerts/default-alerts.png)
</div>
<br></br>
For monitoring critical operations, like a sign-in mutation or
home page query, you can add an operation-specific alert and choose
from the same triggers.

<div style="text-align:center">
![Single Operation](../img/alerts/single-operation.png)
</div>
<br></br>
Now that you have set up alerting, you can add [daily Slack
reports](../integrations/slack.html) or enable the [Datadog
integration](../integrations/datadog.html) to view the Engine data directly in your Datadog dashboard. If you
would like to suggest further features or integrations, please email us at <a href="support@apollographql.com">support@apollographql.com</a>!
