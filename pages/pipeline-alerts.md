# Alerts for failures

If you would like, we can configure alerts to the pipeline yaml file that will send you an email on conditions.

```yaml
stages: {}
notification:
  recipients:
  - recipient: "test@example.com"
    notifier: "c-hmqnj:n-xv76m"
  # Select which statuses you want the notification to be sent
  condition: ["Failed", "Success", "Changed"]
  # Ability to override the default message (Optional)
  message: "my-message"

```