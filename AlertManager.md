## AlertManager

Test SMTP
```
curl -XPOST http://<ALERTMANAGER_SERVICE>:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "SMTPTest",
      "severity": "critical",
      "namespace": "monitoring"
    },
    "annotations": {
      "summary": "Manual SMTP test",
      "description": "Testing Alertmanager email delivery"
    }
  }]'
```
Use wget from with in a pod
```
wget -qO- --post-data='[{
  "labels": {
    "alertname": "SMTPTest",
    "severity": "critical",
    "namespace": "monitoring"
  },
  "annotations": {
    "summary": "SMTP test",
    "description": "Testing email from inside the pod"
  }
}]' \
--header='Content-Type: application/json' \
http://localhost:9093/api/v2/alerts
```
