# Docker + kubectl + KIND Setup

This guide installs and explains three tools needed to work with a local Kubernetes cluster:

* **Docker** — runs containers
* **kubectl** — communicates with Kubernetes
* **KIND** — creates a Kubernetes cluster using Docker containers

The goal is not just to install them. This README explains **what each command does, why we use it, and what happens behind the scenes**.

---

# 1. How the Three Tools Work Together

Before installing anything, understand the big picture.

```text
                    Kubernetes Cluster
                           ▲
                           │
                       kubectl
                           │
                           │ communicates with
                           ▼
                     Kubernetes API
                           ▲
                           │
                         KIND
                           │
                           │ creates
                           ▼
                  Docker Containers
```

### Docker

Docker runs containers.

Example:

```bash
docker run nginx
```

Docker creates and runs an Nginx container.

### KIND

KIND means:

```text
Kubernetes IN Docker
```

KIND uses Docker containers as Kubernetes nodes.

So instead of creating real virtual machines, KIND can create something like:

```text
Docker
  │
  ├── Kubernetes control-plane container
  ├── Kubernetes worker container
  └── Kubernetes worker container
```

Together, these containers form a Kubernetes cluster.

### kubectl

`kubectl` is the command-line tool we use to communicate with Kubernetes.

For example:

```bash
kubectl get nodes
```

means:

> Ask the Kubernetes cluster: "Show me the nodes."

---

# 2. Complete Installation Script

```bash
#!/bin/bash

set -e

echo "===== Updating packages ====="
sudo apt-get update

echo "===== Installing prerequisites ====="
sudo apt-get install -y ca-certificates curl

# --------------------------------------------------
# Docker
# --------------------------------------------------

echo "===== Installing Docker ====="

sudo apt-get install -y docker.io

sudo systemctl enable --now docker

# Add current user to docker group
sudo usermod -aG docker "$USER"

echo "Docker installed."

# --------------------------------------------------
# kubectl
# --------------------------------------------------

echo "===== Installing kubectl ====="

KUBECTL_VERSION="v1.36.0"

curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/kubectl

echo "kubectl installed."

# --------------------------------------------------
# KIND
# --------------------------------------------------

echo "===== Installing KIND ====="

KIND_VERSION="v0.32.0"

curl -Lo kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"

chmod +x kind

sudo mv kind /usr/local/bin/kind

echo "KIND installed."

# --------------------------------------------------
# Verification
# --------------------------------------------------

echo
echo "===== Versions ====="

docker --version
kubectl version --client
kind version

echo
echo "===== Setup completed ====="

echo
echo "IMPORTANT:"
echo "Docker group membership has been updated."
echo "Please log out and log back in."
echo "Then open a terminal and run:"
echo "docker ps"
```

---

# 3. `#!/bin/bash`

```bash
#!/bin/bash
```

This is called a **shebang**.

It tells Linux:

> Run this script using Bash.

The parts are:

```text
#!          → tells Linux this line specifies the interpreter

/bin/bash   → the Bash program
```

So:

```bash
#!/bin/bash
```

means:

```text
Use Bash to execute this file.
```

You can also run a script explicitly with:

```bash
bash script.sh
```

In that case, you are directly telling Bash to execute the script.

---

# 4. `set -e`

```bash
set -e
```

This tells Bash:

> Stop the script if a command fails.

For example:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

Suppose this fails:

```bash
sudo apt-get install -y docker.io
```

Because we have:

```bash
set -e
```

the script stops there.

It does not continue blindly to the next steps.

### Why is that useful?

This is an installation script.

If Docker installation failed, we don't want the script to continue assuming Docker exists.

---

# 5. `echo`

Example:

```bash
echo "===== Updating packages ====="
```

`echo` simply **prints text on the terminal**.

It does not perform the action.

For example:

```bash
echo "Hello"
```

prints:

```text
Hello
```

It does not execute `Hello`.

---

## Why do we use `echo` in this script?

We use it as a label for humans.

For example:

```bash
echo "===== Installing Docker ====="

sudo apt-get install -y docker.io
```

The `echo` tells us:

```text
I am now starting the Docker installation.
```

Then the next command actually performs the installation.

So think:

```text
echo
 ↓
Message for human

actual command
 ↓
Action performed by computer
```

---

## `echo` with nothing

We sometimes write:

```bash
echo
```

This simply prints a blank line.

For example:

```bash
echo "Hello"
echo
echo "World"
```

produces:

```text
Hello

World
```

We use it only to make the output easier to read.

---

# 6. `sudo`

Example:

```bash
sudo apt-get update
```

`sudo` means:

> Run this command with administrator privileges.

Linux protects important system resources.

Installing packages and modifying system files usually requires elevated privileges.

For example:

```bash
apt-get install docker.io
```

may fail for a normal user.

So we use:

```bash
sudo apt-get install docker.io
```

---

# 7. `apt-get`

`apt-get` is Ubuntu/Debian's package-management command.

It is used to:

* update package information
* install packages
* remove packages
* manage packages

---

# 8. `apt-get update`

```bash
sudo apt-get update
```

This **does not install Docker**.

It updates the local package information.

Think about it like this:

```text
Ubuntu package repositories
          ↓
    "What packages
     are available?"
          ↓
Local package information
      gets updated
```

We normally do this before installing packages so that the package manager has current repository information.

---

# 9. `apt-get install`

Example:

```bash
sudo apt-get install -y ca-certificates curl
```

This installs packages.

In this case:

```text
ca-certificates
curl
```

are installed.

---

## What is `-y`?

```bash
-y
```

means:

> Automatically answer "Yes" to installation prompts.

Without `-y`, you may see:

```text
Do you want to continue? [Y/n]
```

The script would then wait for your answer.

With:

```bash
-y
```

it automatically accepts the normal confirmation.

---

# 10. `ca-certificates`

```bash
ca-certificates
```

This installs trusted Certificate Authority certificates.

They are used when Linux communicates securely with HTTPS websites.

Our script downloads Kubernetes and KIND binaries from HTTPS URLs such as:

```text
https://dl.k8s.io/...
https://kind.sigs.k8s.io/...
```

So we install the certificate package as a prerequisite.

---

# 11. `curl`

```bash
curl
```

`curl` is a command-line tool that communicates with URLs.

For example:

```bash
curl https://example.com
```

It can retrieve data from a website.

In our script, we mainly use it to **download files**.

We download:

```text
kubectl
kind
```

---

# Docker Installation

# 12. Install Docker

```bash
sudo apt-get install -y docker.io
```

This tells Ubuntu:

```text
sudo
  ↓
Run with administrator privileges

apt-get
  ↓
Use the package manager

install
  ↓
Install something

-y
  ↓
Automatically answer yes

docker.io
  ↓
Install the Docker package
```

---

# 13. `systemctl`

```bash
sudo systemctl enable --now docker
```

`systemctl` is used to manage Linux services.

Docker runs as a system service.

---

## `enable`

```bash
sudo systemctl enable docker
```

means:

> Configure Docker to start automatically when the machine boots.

---

## `--now`

```bash
sudo systemctl enable --now docker
```

`--now` means:

> Start the service immediately too.

So this one command does two things:

```text
1. Start Docker now
2. Enable Docker to start automatically after boot
```

Without `--now`, you could use:

```bash
sudo systemctl start docker
```

to start Docker manually.

---

# 14. Add the User to the Docker Group

```bash
sudo usermod -aG docker "$USER"
```

This is an important command.

It adds the current user to the Linux `docker` group.

---

## Why do we do this?

Docker has a background service called the **Docker daemon**.

The Docker client communicates with that daemon.

The Docker daemon's Unix socket is protected, so normal users may not be allowed to access it directly.

After adding the user to the `docker` group, the user can normally run:

```bash
docker ps
```

instead of:

```bash
sudo docker ps
```

---

# 15. Breaking Down `usermod -aG`

The command is:

```bash
sudo usermod -aG docker "$USER"
```

Let's split it:

```text
sudo
 ↓
Run as administrator

usermod
 ↓
Modify an existing Linux user

-a
 ↓
Append the group

-G
 ↓
Supplementary groups

docker
 ↓
The group we are adding

"$USER"
 ↓
The current username
```

---

## What is `$USER`?

`$USER` is a shell variable containing the current username.

For example, suppose your username is:

```text
shaad
```

Then:

```bash
echo "$USER"
```

would produce:

```text
shaad
```

So:

```bash
sudo usermod -aG docker "$USER"
```

effectively becomes:

```bash
sudo usermod -aG docker shaad
```

---

# 16. Why Don't We Create a `kubectl` or `kind` Group?

We don't need:

```text
kubectl group
kind group
```

just to execute those commands.

Why?

Because `kubectl` and `kind` are executable programs.

We install them into:

```text
/usr/local/bin/
```

So:

```text
kubectl
   ↓
Executable program

kind
   ↓
Executable program
```

Docker is different.

```text
docker command
      ↓
Docker daemon
      ↓
Protected Unix socket
      ↓
Permissions matter
```

That is why the Docker group is relevant.

---

# kubectl Installation

# 17. Set the kubectl Version

```bash
KUBECTL_VERSION="v1.36.0"
```

This creates a shell variable.

The variable is:

```text
KUBECTL_VERSION
```

and its value is:

```text
v1.36.0
```

Later we use:

```bash
${KUBECTL_VERSION}
```

which gets replaced by:

```text
v1.36.0
```

So this:

```bash
https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl
```

becomes:

```text
https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl
```

---

# 18. Download kubectl

```bash
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
```

This downloads the `kubectl` binary.

Let's understand the options.

---

## `-L`

```bash
-L
```

means:

> Follow redirects.

Sometimes a URL sends us to another URL.

`curl -L` follows that redirect.

---

## `-O`

```bash
-O
```

means:

> Save the downloaded file using the filename from the URL.

The URL ends with:

```text
/kubectl
```

Therefore the saved file is:

```text
kubectl
```

So:

```bash
curl -LO URL
```

can be understood as:

```text
Download the file
      ↓
Follow redirects
      ↓
Use the filename from the URL
      ↓
Create:
kubectl
```

---

# 19. `chmod +x`

```bash
chmod +x kubectl
```

`chmod` means:

> Change file permissions.

`+x` means:

> Add execute permission.

We downloaded:

```text
kubectl
```

It is a file.

We want Linux to treat it as an executable program.

So we run:

```bash
chmod +x kubectl
```

Now it can be executed.

---

# 20. Move kubectl to `/usr/local/bin`

```bash
sudo mv kubectl /usr/local/bin/kubectl
```

`mv` means:

> Move a file.

We move:

```text
kubectl
```

from the current directory to:

```text
/usr/local/bin/kubectl
```

---

## Why `/usr/local/bin`?

`/usr/local/bin` is commonly used for programs installed manually.

It is normally included in the system `PATH`.

Therefore we can type:

```bash
kubectl
```

instead of:

```bash
/usr/local/bin/kubectl
```

---

# KIND Installation

# 21. Set the KIND Version

```bash
KIND_VERSION="v0.32.0"
```

This creates:

```text
KIND_VERSION
      ↓
v0.32.0
```

We later use the variable inside the download URL.

---

# 22. Download KIND

```bash
curl -Lo kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"
```

This downloads the KIND binary.

---

## `-L`

Same meaning as before:

```bash
-L
```

means:

> Follow redirects.

---

## `-o kind`

Lowercase `-o` means:

> Save the downloaded file using the filename I provide.

So:

```bash
-o kind
```

creates:

```text
kind
```

---

## Why is this different from kubectl?

For kubectl we used:

```bash
curl -LO URL
```

Here we use:

```bash
curl -Lo kind URL
```

The difference is:

```text
-O
 ↓
Use filename from URL

-o kind
 ↓
I choose the filename:
kind
```

---

# 23. Make KIND Executable

```bash
chmod +x kind
```

Again:

```text
chmod
 ↓
Change permissions

+x
 ↓
Add execute permission
```

Now Linux can execute the KIND program.

---

# 24. Move KIND to `/usr/local/bin`

```bash
sudo mv kind /usr/local/bin/kind
```

Move the KIND executable to:

```text
/usr/local/bin/kind
```

Because `/usr/local/bin` is normally in `PATH`, we can later simply type:

```bash
kind
```

---

# Verification

# 25. Check Docker Version

```bash
docker --version
```

This displays the installed Docker version.

`--version` means:

> Show me the version.

---

# 26. Check kubectl Version

```bash
kubectl version --client
```

The important part here is:

```bash
--client
```

This tells kubectl:

> Show information about the kubectl client itself.

We can therefore check kubectl even when a Kubernetes cluster has not been created yet.

---

# 27. Check KIND Version

```bash
kind version
```

This displays the installed KIND version.

---

# 28. Why Do We Verify Versions?

We installed several programs.

Before moving on, we want to check:

```text
Docker installed?
       ↓
kubectl installed?
       ↓
KIND installed?
```

The version commands give us a quick confirmation.

---

# 29. What Happens After the Script Finishes?

The script prints:

```bash
echo
echo "===== Setup completed ====="
echo
echo "IMPORTANT:"
echo "Docker group membership has been updated."
echo "Please log out and log back in."
echo "Then open a terminal and run:"
echo "docker ps"
```

This part does **not install anything**.

It simply gives instructions to the person running the script.

---

# 30. Why Do We Need to Log Out and Log Back In?

Earlier we ran:

```bash
sudo usermod -aG docker "$USER"
```

This changed the user's group membership.

However, your **current login session was created before that change**.

The current shell therefore does not automatically receive the new group membership.

To start a fresh login session:

```text
Current session
      ↓
Log out
      ↓
Log in again
      ↓
New session
      ↓
New docker group membership is available
```

So the README asks you to:

### Step 1

Log out of your Ubuntu session.

### Step 2

Log in again.

### Step 3

Open a terminal.

### Step 4

Run:

```bash
docker ps
```

---

# 31. Why `docker ps`?

```bash
docker ps
```

asks Docker:

> Show me the currently running containers.

If nothing is running, you may see a table with no containers.

That is completely fine.

For example:

```text
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

with no rows underneath means:

> Docker is running, but currently there are no running containers.

---

# 32. Why Don't We Use `sudo docker ps`?

We added the user to the Docker group:

```bash
sudo usermod -aG docker "$USER"
```

The purpose is to allow normal Docker usage without repeatedly writing `sudo`.

So after logging in again, we want:

```bash
docker ps
```

not:

```bash
sudo docker ps
```

---

# 33. Complete Setup Flow

Keep this picture in your mind:

```text
Ubuntu
  │
  ├── apt-get update
  │       ↓
  │   Update package information
  │
  ├── Install prerequisites
  │       ├── ca-certificates
  │       └── curl
  │
  ├── Install Docker
  │       ↓
  │   Start Docker service
  │       ↓
  │   Add user to docker group
  │
  ├── Install kubectl
  │       ↓
  │   Download binary
  │       ↓
  │   chmod +x
  │       ↓
  │   /usr/local/bin/kubectl
  │
  └── Install KIND
          ↓
      Download binary
          ↓
      chmod +x
          ↓
      /usr/local/bin/kind
```

---

# 34. What We Have After Installation

After the installation:

```text
Docker
  ↓
Container runtime
```

```text
KIND
  ↓
Uses Docker
  ↓
Creates Kubernetes nodes
  ↓
Creates a local Kubernetes cluster
```

```text
kubectl
  ↓
Communicates with Kubernetes
  ↓
Controls the cluster
```

So the overall relationship is:

```text
                 kubectl
                    │
                    │
                    ▼
            Kubernetes API
                    │
                    │
                  KIND
                    │
                    ▼
             Docker containers
                    │
                    ▼
             Kubernetes nodes
```

---

# 35. Final Checklist

After running the installation script:

* Docker should be installed.
* Docker service should be running.
* Your user should be added to the `docker` group.
* kubectl should be available as `kubectl`.
* KIND should be available as `kind`.
* You should log out and log back in after the Docker group change.
* Then run:

```bash
docker --version
kubectl version --client
kind version
docker ps
```

If these work correctly, your basic Docker + kubectl + KIND environment is ready for creating a local Kubernetes cluster.

---

# 36. The Most Important Things to Remember

You do **not** need to memorize every command immediately.

Understand these relationships first:

```text
apt-get
  ↓
Installs Linux packages
```

```text
curl
  ↓
Downloads files from URLs
```

```text
chmod +x
  ↓
Makes a file executable
```

```text
/usr/local/bin
  ↓
Place for manually installed executable programs
```

```text
systemctl
  ↓
Controls Linux services
```

```text
usermod -aG docker "$USER"
  ↓
Adds your user to Docker group
```

```text
Docker
  ↓
Runs containers
```

```text
KIND
  ↓
Uses Docker to create a Kubernetes cluster
```

```text
kubectl
  ↓
Communicates with Kubernetes
```

Once these relationships are clear, the entire installation script becomes much easier to understand.
