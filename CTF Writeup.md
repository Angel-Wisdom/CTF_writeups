

CTF Writeup: Who's Really Invincible?? (200pts)
Category: OSINT Points: 200

## Challenge Description
We were given an image of a gaming setup and tasked with finding the flag hidden somewhere
online.
## Solution
Step 1 — Identify the Username from the Image
Looking at the gaming setup photo (Image 1), the username whitedev4444 was clearly visible in
multiple places — on the neon sign, the gaming chair, and the PC case. This was the starting point.

Step 2 — Find the Instagram Profile
Searching for whitedev4444 on Instagram led to their profile. Browsing through their posts, one
caption stood out:


"Hello there ;))) Posting on an interface which originated 21 years ago btw...."
And on another post, a comment read:
"I like the content devop1938 posts :)"
Two crucial breadcrumbs in one place.
Step 3 — Decode the Hint
"interface which originated 21 years ago"
Counting back 21 years from 2025 → 2004 — the year YouTube was founded. This pointed directly
to YouTube.








Step 4 — Find the YouTube Channel

Searching for devop1938 on YouTube led to the channel Mrwhitedev. Checking the channel's
About/Description section revealed:
u found the flag-TCHNVTE{I_l1k3_my_g4ll3ry}
## Flag
TCHNVTE{I_l1k3_my_g4ll3ry}
