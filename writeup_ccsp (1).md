

CCSP Writeup
- Register and Login as a Normal User
- Begin by registering a new account and logging into the web application.
- After login, you are presented with a dashboard and options to access Classified/Admin
## Resources.
- Inspect API Requests & User Discovery
- Browse the dashboard and intercept requests using a proxy tool (e.g., Burp Suite).
- Visit your profile or account info page to see your user data, including an id, role,
and username.
- Observe that user IDs appear to be hashed (MD5 format).

- Identify the Hash Mechanism
- By comparing the test data, recognize that the id field uses MD5 hashes.
- admin in MD5 is 21232f297a57a5a743894a0e4a801fc3.


- Extract the Required Admin Details
- The details shown for admin might resemble:
## {
## "id": "21232f297a57a5a743894a0e4a801fc3",
## "role": "captainoftheship",
## "username": "admin"
## }
- Exploit: Register as Admin Using a Custom Request
- Use your proxy/interceptor to modify the registration request.
- In the POST request to the registration API (e.g., /register), add or override the parameters:
## "id": "21232f297a57a5a743894a0e4a801fc3",
## "role": "captainoftheship"
- Fill the rest of the registration as normal (choose any username/password).
- Access the Flag
- After registering, login as the new user with the elevated role.
- Go to the flag page:
http://127.0.0.1:5003/flag
- With role: captainoftheship and correct or spoofed admin hash, you will successfully access
the flag.
FLAG{API_0bfu5c4t10n_D03snt_St0p_M3}
