### Logging

Logging is where you record and store machine data for x number of days, that you can submit as evidence for incident response.
* If you have AD start with Windows Event Forwarding (WEF).  Enable Windows Advanced Audit Logging with GP for all servers and workstations connected to your domain.
* If you have resources, you can go a step further and look at the Elasticsearch stack or Splunk, and you can not just store your logs, but go and do further analysis, get notifications for security events and even trigger automated remediation steps.  But HEY! Warning!, this is not essential if you don’t have the basics outlined in this article in place first.