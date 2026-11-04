# Deploy MongoDB on an Ubuntu EC2 instance


## 1. Prepare the instance and connect

Use your course's AWS account, region, instance size, and `DataStudents` key pair. Select an Ubuntu x86_64 AMI. The instance needs outbound package-repository access. For SSH, add an inbound TCP **22** rule whose source is **My IP** (your public IPv4 address with `/32`). Keep port 27017 closed for the initial setup.

On your **Mac**, put the private key in `~/.ssh`, then restrict its permissions and connect. Replace `YOUR_EC2_PUBLIC_DNS` throughout this guide with the instance's actual public DNS name.

```bash
chmod 400 ~/.ssh/DataStudents.pem
ssh -i ~/.ssh/DataStudents.pem ubuntu@YOUR_EC2_PUBLIC_DNS
```

On the **Ubuntu instance**, confirm the OS and architecture:

```bash
cat /etc/os-release
dpkg --print-architecture
```

Expect Ubuntu 24.04 (`noble`) or 22.04 (`jammy`), and `amd64`. This script deliberately excludes ARM/Graviton, other Ubuntu releases, Amazon Linux and macOS.

## 2. Deploy using the provisioning script

Save `mongo_setup.sh` alongside this README on your Mac. In a **Mac terminal opened in that folder**, copy it to the instance:

```bash
scp -i ~/.ssh/DataStudents.pem mongo_setup.sh ubuntu@YOUR_EC2_PUBLIC_DNS:~/
```

In your **Ubuntu SSH session**, run:

```bash
chmod +x ~/mongo_setup.sh
sudo ~/mongo_setup.sh
```

The script validates the host, installs the signing key and matching repository, installs MongoDB, enables its service, and waits for a successful database ping. An error stops the script with a nonzero exit code. Re-running on an existing 9.0 installation preserves packages, configuration and data; other installed server versions are rejected.

The initial database accepts local connections without authentication. Use a dedicated training instance with trusted local users. Complete section 5 before granting remote database access. No AWS rules or MongoDB bind settings are changed by the script.

## 3. Manual installation alternative

These commands explain the installation stages. Use this route **instead of** running the script on a fresh supported Ubuntu instance.

### Install tools and trust the package signing key

Run on **Ubuntu**:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
curl -fsSL https://pgp.mongodb.com/server-9.asc -o /tmp/mongodb-server-9.asc
sudo gpg --batch --yes --dearmor \
  --output /usr/share/keyrings/mongodb-server-9.gpg /tmp/mongodb-server-9.asc
sudo chmod 644 /usr/share/keyrings/mongodb-server-9.gpg
```

### Add the repository matching Ubuntu

```bash
. /etc/os-release
case "$VERSION_ID:$VERSION_CODENAME" in
  22.04:jammy|24.04:noble) ;;
  *) echo "Unsupported Ubuntu release"; exit 1 ;;
esac
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg] https://repo.mongodb.org/apt/ubuntu ${VERSION_CODENAME}/mongodb-org/9.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
sudo apt-get update
```

### Install and start the database

```bash
sudo apt-get install -y mongodb-org
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
```

Use MongoDB's `mongodb-org` package. If the unofficial Ubuntu `mongodb` package is already installed, use a fresh instance for this exercise rather than mixing packages. These steps follow the [official MongoDB Ubuntu installation instructions](https://www.mongodb.com/docs/manual/administration/install-community-linux/?linux-distro=ubuntu&linux-method=pkg).

## 4. Verify and try a document

Run on **Ubuntu**:

```bash
sudo systemctl status mongod --no-pager
sudo systemctl is-enabled mongod
mongosh --quiet --eval 'db.adminCommand({ping: 1})'
sudo ss -lntp | grep 27017
```

Expect `active (running)`, `enabled`, a ping result containing `ok: 1`, and a listener on `127.0.0.1:27017`. These are expected results to check on the instance, not results captured from a live deployment.

Open `mongosh` and enter these **MongoDB shell** commands:

```javascript
use training
db.deployment_check.updateOne(
  { name: "mongodb-ec2-lab" },
  { $set: { status: "working" } },
  { upsert: true }
)
db.deployment_check.findOne({ name: "mongodb-ec2-lab" })
exit
```

The upsert creates or updates one sample document; the query should return it.

Useful paths: `/etc/mongod.conf` (configuration), `/var/lib/mongodb` (data) and `/var/log/mongodb/mongod.log` (log). Restart the service after editing its configuration.

## 5. Add authentication before remote access

While MongoDB is still bound only to localhost, run `mongosh` **on Ubuntu**. Create a user administrator and a separate user limited to the training database. Choose different strong passwords when prompted; do not put passwords in scripts or connection URLs.

```javascript
use admin
db.createUser({
  user: "labAdmin",
  pwd: passwordPrompt(),
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }]
})
use training
db.createUser({
  user: "trainingUser",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "training" }]
})
exit
```

Back in the **Ubuntu terminal**, back up and edit the configuration:

```bash
sudo cp -p /etc/mongod.conf /etc/mongod.conf.before-auth
sudo nano /etc/mongod.conf
```

Add or update the single top-level `security` section using spaces:

```yaml
security:
  authorization: enabled
```

Save, restart, and test a protected read:

```bash
sudo systemctl restart mongod
mongosh --host 127.0.0.1 --port 27017 --username trainingUser \
  --authenticationDatabase training --password
```

After entering the training user's password, run `use training`, then `db.deployment_check.findOne()`. A new unauthenticated shell should be denied that read. A successful ping alone does not prove authentication is enforced.

This follows MongoDB's [SCRAM authentication procedure](https://www.mongodb.com/docs/manual/tutorial/configure-scram-client-authentication/).

## 6. Connect from your Mac through SSH

For an encrypted lab connection, leave `bindIp: 127.0.0.1` and keep inbound 27017 closed. Run this in a **new Mac terminal** and leave it open:

```bash
ssh -i ~/.ssh/DataStudents.pem -N -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:27018:127.0.0.1:27017 ubuntu@YOUR_EC2_PUBLIC_DNS
```

In MongoDB Compass on your Mac, use:

```text
mongodb://127.0.0.1:27018/training?authSource=training&directConnection=true
```

Select **Username / Password** authentication, username `trainingUser`, authentication database `training`, and enter the password in Compass's password field. Connect and inspect `training.deployment_check`. Port 27018 avoids colliding with a MongoDB server already on your Mac. Close the SSH tunnel with Ctrl+C when finished.

## 7. Extra task: allow direct MongoDB access on port 27017

This route demonstrates both requested changes. Complete authentication first. For an internet connection, configure TLS with a trusted certificate before opening the inbound rule; use the SSH route above until certificates are available. Direct private-network connections should also use TLS.

### Update mongod.conf

Get the database instance's **private IPv4 address** from EC2 instance details. Back up `/etc/mongod.conf`, then edit its existing `net` section. Replace `YOUR_EC2_PRIVATE_IPV4` below. Keep the other configuration sections, including `security.authorization: enabled`; do not add a duplicate `net` section.

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,YOUR_EC2_PRIVATE_IPV4
```

EC2's public IPv4 is mapped to the instance's private interface, so bind to the private address. MongoDB recommends [binding to trusted private interfaces](https://www.mongodb.com/docs/manual/core/security-mongodb-configuration/).

For direct TLS access, obtain a CA-signed certificate whose subject alternative name matches the DNS name clients use. Install its combined certificate/private-key PEM readable only by the `mongodb` service account and root. Also install the trusted CA certificate bundle, readable by `mongodb`. Merge this under the same `net` section, replacing both paths with the actual files:

```yaml
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: true
```

`allowConnectionsWithoutCertificates: true` permits password-authenticated TLS clients without requiring a client certificate; `security.authorization: enabled` still enforces database authentication. It does not disable client-side validation of the server certificate. These settings follow the [MongoDB TLS configuration options](https://www.mongodb.com/docs/manual/reference/configuration-options/#net.tls-options). Follow the [official TLS setup](https://www.mongodb.com/docs/manual/tutorial/configure-ssl/) for certificate requirements. Restart and inspect the service before changing AWS rules:

```bash
sudo systemctl restart mongod
sudo systemctl status mongod --no-pager
sudo ss -lntp | grep 27017
```

### Add the EC2 inbound rule

In AWS: **EC2 → Instances → select your instance → Security → security group → Edit inbound rules → Add rule**.

| Field | Value |
| --- | --- |
| Type | Custom TCP |
| Protocol | TCP |
| Port range | 27017 |
| Source for your Mac | My IP (your current public IPv4 `/32`) |
| Source for an app in the VPC | The app instance's security group, instead of My IP |
| Description | MongoDB training access from approved client |

Save the rule. Keep any host firewall restricted to the same approved source if it is enabled. Never use `0.0.0.0/0` or `::/0` for database access. AWS describes [database security group sources](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html#database-server-rules) as specific CIDRs or application security groups.

In Compass, use the certificate-matching DNS name and port 27017 with TLS enabled:

```text
mongodb://YOUR_CERTIFICATE_MATCHING_DNS:27017/training?authSource=training&directConnection=true&tls=true
```

Enter `trainingUser` and its password in the authentication fields. If using an organizational CA, supply its CA certificate in the TLS settings. Keep hostname and certificate validation enabled. Confirm a document read from the approved client; a client outside the permitted source should not connect.

After enabling `requireTLS`, local `mongosh` connections also need TLS options and a certificate-matching host. The script's original plain local ping will then fail on a re-run, because it preserves this custom configuration. Use the TLS connection for verification.

## 8. Troubleshoot and finish

| Symptom | Check |
| --- | --- |
| SSH timeout | Instance running, correct DNS/key/region, TCP 22 source matches your current IP |
| `Permission denied (publickey)` | SSH username is `ubuntu`; key pair matches; key mode is 400 |
| Package not found | Repository codename matches Ubuntu; `sudo apt-get update` succeeds |
| Service fails | `sudo journalctl -u mongod -n 50 --no-pager` and `sudo tail -n 50 /var/log/mongodb/mongod.log`; inspect YAML indentation |
| Database connection refused | Service running, correct host/port, expected listener from `ss` |
| Remote timeout | Correct security group, approved source IP, private bind address, routes and host firewall |
| Authentication failed | Correct username, password and authentication database (`training`) |
| TLS validation failed | DNS matches certificate SAN, correct CA, valid dates and readable server PEM |

Record the script output, service status, successful document read, and any optional Compass connection as deployment evidence. This folder contains the prepared files; it does not claim an EC2 deployment was executed.

When the exercise is finished, preserve any data you need, close the tunnel and remove the temporary 27017 rule. Follow the trainer's reminder to terminate instances you no longer need; termination can destroy attached data, so check before doing it. Do not leave an unused training instance running.
