# RabbitMQ — Troubleshooting

## Entry template

**Problem:**  
**Symptom:**  
**Cause:**  
**Solution:**  

---

## Entries

### 001 — User cannot log in to Management UI

**Problem:** Created a new user and shared the credentials with the team, but they cannot log in to the UI on port 15672.

**Symptom:** The login page returns `Login failed` even with the correct username and password.

**Cause:** A new user is created without any tags. Without the `management` tag RabbitMQ denies access to the UI, even if the user exists and the password is correct.

**Solution:**
```bash
docker exec rabbitmq rabbitmqctl set_user_tags <username> management
```

Verify the tag is set:
```bash
docker exec rabbitmq rabbitmqctl list_users
# user         tags
# myuser       [management]  ← should look like this
```
