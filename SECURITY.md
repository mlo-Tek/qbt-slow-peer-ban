# Security notes

Do not commit a filled-in `.env` file or a compose file containing real credentials, internal IP addresses or other private configuration. Use `.env.example` as the template; `.env` is git-ignored.

If credentials have previously been committed to GitHub, removing them from the latest file is not enough:
rotate the credentials and remove the secret from Git history as well.
