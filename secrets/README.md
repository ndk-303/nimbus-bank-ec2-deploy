# Firebase credential

Copy the Firebase Admin service-account JSON to `firebase-admin.json` in this directory on the EC2 host only. This file is ignored by Git and is not included in this repository. The backend container runs as UID 10001 and must be able to read it; keep it inaccessible to unrelated users.
