# class-notes-website
# Class Notes Hub

A small website where students pick their semester and download lecture notes for each unit. The web page runs on an EC2 server inside a VPC, and the PDF notes are stored in an S3 bucket.

## The problem

Students often struggle to find lecture notes. They are scattered across chats, emails and drives, and it is hard to know which file belongs to which unit. Class Notes Hub puts all the notes in one place with a simple page: choose Semester 1 or 2, then download the notes for each unit.

## Architecture

```
Student's browser
      |
      |  HTTP (port 80)
      v
Internet Gateway --> Public subnet (in my VPC)
                        |
                        +--> EC2 instance running Apache (serves index.html)
                                   |
                                   |  links to PDFs (HTTPS)
                                   v
                        S3 bucket: class-notes-hub-04  (public/ prefix is readable)
```

- **VPC:** [notes-vpc-vanessa / CIDR, e.g. 10.0.0.0/16]
- **Public subnet:** [notes-public-subnet]. Its route table sends `0.0.0.0/0` to the internet gateway, so the web server is reachable.
- **Private subnet:** [notes-private-subnet]. It has no route to the internet gateway, so it is meant for resources that should not be exposed, such as databases.
- **EC2:** [ t3.micro] running Amazon Linux with Apache (`httpd`), set up automatically by a user data script.
- **S3:** bucket `class-notes-hub-04` in `eu-north-1`. Notes are stored under `public/`.
- **Security group:** HTTP (80) from anywhere, SSH (22) from My IP only.

## Steps I followed

1. **Created the VPC and subnets** and attached an internet gateway. Added a route `0.0.0.0/0` to the gateway in the public subnet's route table.![VPC and subnets](docs/screenshots/Screenshot%202026-11-15%20200324.png)
2. **Created the S3 bucket** `class-notes-hub-04` and uploaded the PDFs into a `public/` folder.![S3 Bucket](docs/screenshots/Screenshot%202026-11-15%20185252.png) 
3. **Turned off Block Public Access** for the bucket and added a bucket policy that allows `s3:GetObject` only on `public/*`. 
4. **Tested a PDF link directly** in the browser to confirm S3 access worked.![PDF Test](docs/screenshots/Screenshot%202026-11-17%20000956.png)
5. **Created a security group** allowing HTTP from anywhere and SSH from my IP only. ![security group](docs/screenshots/Screenshot%202026-11-15%20201604.png)
6. **Launched the EC2 instance** in the public subnet with auto-assign public IP enabled and the user data script (`user-data.sh`).
7. **Opened `public ip`** and tested every download button. !Instance was terminated to avoid accumulating costs.
8. **Cleaned up** by terminating the instance when finished. ![Terminated instance]

### Bucket policy used

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadNotesOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::class-notes-hub-04/public/*"
    }
  ]
}
```

## Why we combined the services

1. **Why are the PDFs in S3 and not on the EC2 server?**
   S3 is cheap, durable and built for storing files, and it scales without any server work. EC2 only has to serve one small HTML page. If the server is replaced, the notes stay safe.

2. **What makes a subnet public, and what is the private subnet for?**
   A subnet is public when its route table has a route to an internet gateway, so resources with public IPs can be reached from the internet. A private subnet has no such route and is for things that should not be exposed, such as databases.

3. **Why is SSH allowed only from My IP, but HTTP from anywhere?**
   Every student needs to open the website, so HTTP must be open to everyone. Administrative access should be limited to me, which greatly reduces the chance of attacks on the server.

4. **Why does the bucket policy use `public/*` and not the whole bucket?**
   This follows least privilege. Only the notes are public, and anything else I store in the bucket stays private.

5. **Why is user data better than typing the setup commands by hand?**
   It runs automatically at launch, so every instance is set up the same way with no typing mistakes. It saves time and makes the setup repeatable, for example when I had to terminate and relaunch.

## Problems I hit and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| Clicking a download link showed an XML `AccessDenied` error | The file names in my web page (e.g. `Calculus-I-Week-1-Notes.pdf`) did not match the files in S3 (`week-1-notes.pdf`). S3 returns AccessDenied instead of "not found" because the policy does not allow listing. | Updated the `units` list in `index.html` and `user-data.sh` to the exact file names in `public/`. |
| `This site can't be reached` on the EC2 public IP | The security group did not allow HTTP on port 80 from the internet, and I was using the wrong group. | Created a new security group with HTTP (80) from anywhere and SSH from My IP. |
| `Key pair already exists` when relaunching | Key pairs are not deleted when an instance is terminated. | Selected the existing key pair from the dropdown instead of creating a new one. |
| Only the `default` security group was available | The launch screen was set to a different VPC than the one the group belonged to. | Selected the correct VPC and created a new security group in it. |
| User data script might not run | `#!/bin/bash` must be alone on the first line and `EOF` alone on the last line. | Put each command on its own line and checked both lines. |
| `download` attribute ignored on links | Browsers ignore it for links to a different origin (EC2 page linking to S3). | Optional: use `target="_blank"` so PDFs open in a new tab. |

## Proof of clean-up

Screenshot of the terminated instance:Screenshot 2026-11-17 001230.png

## Files in this repo

- `index.html`: the web page
- `user-data.sh`: EC2 user data script that installs Apache and creates the page
- `.gitignore`: blocks `*.pem` key files from being uploaded
- `docs/screenshots/`: screenshots used in this README


