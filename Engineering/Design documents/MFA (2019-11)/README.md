## V1

### Config:

- auth header

Profile:

- Body

## V2

### Config:

```yaml
mode: "single" # or "challenge/verify"
endpoints:
  single: # or challenge or verify
    method: "POST"
    url: "https://sms-api/api/v2/?auth=abc123&phoneNumber={{ phoneNumber }}"
    body: '{"phoneNumber":"{{ phoneNumber }}","message":"Hello, {{ username }} here is your code: {{ token }}"}'
    headers:
      authorization: abc123
      Content-type: application/json
```

### profile.mfa (what is saved in profile upon successful activation)

```yaml
username: 'bob',
phoneNumber: '1234'
```
