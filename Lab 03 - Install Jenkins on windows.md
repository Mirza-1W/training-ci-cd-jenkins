# Jenkins on Windows — Step-by-Step Lab

## Lab Objectives

* Install the Jenkins **LTS** Windows service.
* Unlock Jenkins and create the first admin user.
* Install recommended plugins.
* Configure required tools (Java, Git; optional Maven/Node).
* Verify with a sample job.
* Learn quick fixes for the most common errors.

## Prerequisites

* Windows 10/11 (Pro/Enterprise) or Windows Server 2019/2022.
* Local admin rights on the machine and open network egress (for plugin downloads).
* **Port 8080** available (default Jenkins HTTP).

  > If 8080 is busy, you’ll change it later.

---

## Step 0 — (Recommended) Install Java 17 LTS

Jenkins LTS works best with **Java 17**.

1. Install a Java 17 distribution (e.g., Adoptium Temurin or Microsoft Build of OpenJDK).

2. Verify:

   ```powershell
   java -version
   ```

   You should see `17.x`.

3. (Optional) Set `JAVA_HOME` (helps some tools):

   ```powershell
   # Replace the path with your actual JDK install directory
   setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-17"
   setx PATH "$env:PATH;%JAVA_HOME%\bin"
   ```

   Close and reopen PowerShell, then re-run `java -version`.

> Tip: Jenkins bundles its own JRE in newer MSI installers; having a system JDK is still useful for builds.

---

## Step 1 — Download & Run the Jenkins Windows Installer (MSI)

1. Download the **Jenkins LTS MSI** (Windows installer) from the official site.

2. Double-click the MSI:

   * **Destination folder**: keep default (`C:\Program Files\Jenkins`).
   * **Run service as**:

     * *Quick start*: **Local System** (works for most demos).
     * *Production*: create a dedicated `jenkins` service account with “Log on as a service”.
   * **Port**: keep **8080** (change later if needed).

3. Finish the wizard—this:

   * Creates Windows service **Jenkins**.
   * Sets **JENKINS_HOME** (default `%ProgramData%\Jenkins\.jenkins`).
   * Starts the service.

4. Verify the service:

   ```powershell
   Get-Service Jenkins
   # Status should be Running
   ```

5. If Windows Defender Firewall prompts, allow access for Java/Jenkins on private networks (or add a rule below).

---

## Step 2 — Open Jenkins UI & Unlock

1. Open a browser: **[http://localhost:8080/](http://localhost:8080/)**

2. You’ll see **“Unlock Jenkins”** asking for an initial admin password.

   Retrieve it:

   ```powershell
   Get-Content "C:\Program Files\Jenkins\secrets\initialAdminPassword"
   ```

   > If installed elsewhere, the file also exists at `%JENKINS_HOME%\secrets\initialAdminPassword`.

3. Paste the password in the browser and continue.

---

## Step 3 — Install Plugins & Create Admin User

1. **Install suggested plugins** (recommended to start).
2. Create your **admin user** (username, password, full name, email).
3. Confirm the Jenkins URL (keep default `http://localhost:8080/` for now).
4. Wait for plugin installation to complete → click **Start using Jenkins**.

---

## Step 4 — Add/Verify Build Tools

At **Manage Jenkins → Tools**:

* **Git**

  * Install Git on Windows if missing: [https://git-scm.com/download/win](https://git-scm.com/download/win)
  * After install, verify:

    ```powershell
    git --version
    ```
  * In Jenkins Tools, add *Git* → “Git installations” → Name `Default` → Leave path blank (auto-discovery) or set explicit path like `C:\Program Files\Git\bin\git.exe`.

* **JDK** (if you want Jenkins to manage multiple JDKs)

  * Add a JDK entry (Name: `jdk17`) and uncheck “Install automatically” if you already set `JAVA_HOME`; otherwise configure automatic installer.

* **Optional**:

  * **Maven**: install on Windows and add here, or let Jenkins auto-install.
  * **NodeJS**: install Node and add NodeJS installation; use the NodeJS plugin for per-job PATH.

---

## Step 5 — Create a First Job (Smoke Test)

1. **New Item** → **Freestyle project** → Name: `hello-jenkins`.
2. In **Build** section → **Add build step**:

   * **Windows**: *Execute Windows batch command*
   * Command:

     ```bat
     echo Hello from Jenkins on Windows!
     ```
3. **Save** → **Build Now**.
4. Click the build → **Console Output** → Confirm you see the message and **Finished: SUCCESS**.

---

## Step 6 — (If Needed) Firewall & Port Fixes

### Open firewall for 8080:

```powershell
New-NetFirewallRule -DisplayName "Jenkins 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

### Change Jenkins Port (e.g., to 9090):

1. Stop service:

   ```powershell
   Stop-Service Jenkins
   ```
2. Edit `C:\Program Files\Jenkins\jenkins.xml`
   Find the `--httpPort=8080` argument and change to `--httpPort=9090`.
3. Start service:

   ```powershell
   Start-Service Jenkins
   ```
4. Browse to **[http://localhost:9090/](http://localhost:9090/)**

---

## Step 7 — Service Account & Folder Permissions (Production Tip)

If you run the service under a **dedicated user** (recommended):

1. In **services.msc** → Jenkins → **Log On** tab → set the dedicated account.
2. Grant that account **Read/Write** to `%JENKINS_HOME%` and any workspace/checkout directories (and network shares if used).
3. Restart the service.

---

## Step 8 — Backup & Upgrade Basics

* **Backup**: copy `%JENKINS_HOME%` (jobs, users, plugins, secrets).
* **Upgrade Jenkins**:

  * Download the newer MSI and run **in-place** (keeps config).
  * After upgrade, verify plugin compatibility and core version.
* **Backup before upgrade** is strongly advised.

---

## Step 9 — Troubleshooting Quick Guide

### A) Jenkins page doesn’t load

* Check service:

  ```powershell
  Get-Service Jenkins
  Start-Service Jenkins
  ```
* Check if port is busy:

  ```powershell
  netstat -aon | findstr :8080
  # If occupied, change port in jenkins.xml (Step 6)
  ```

### B) “This site can’t be reached” or plugin downloads fail

* Corporate proxy? Configure at **Manage Jenkins → Manage Plugins → Advanced** (HTTP/HTTPS proxy).
* Ensure outbound access and SSL interception exceptions if applicable.

### C) “Access Denied” to workspace or tools

* If using a dedicated service account, ensure NTFS permissions on:

  * `%JENKINS_HOME%`
  * Your Git workspace/checkout paths
  * Tool paths (Git, Maven, JDK)

### D) Service keeps stopping

* Check logs:

  * `%JENKINS_HOME%\jenkins.log`
  * `%JENKINS_HOME%\jenkins.err.log` / Windows Event Viewer → **Windows Logs → Application**
* Verify Java is present and compatible if using system JDK.

---

## (Optional) Alternative: Run Jenkins via WAR on Windows

Use this if you prefer not to install a Windows service.

1. Download `jenkins.war` (LTS).
2. Run:

   ```powershell
   cd C:\tools\jenkins
   java -jar jenkins.war --httpPort=8080
   ```
3. Open **[http://localhost:8080/](http://localhost:8080/)** and proceed with **Unlock Jenkins** as in Step 2.

   > To run in background, consider **NSSM** or use Windows Task Scheduler.

---

## Validation Checklist

* [ ] `Get-Service Jenkins` shows **Running**
* [ ] Browser opens **[http://localhost:8080/](http://localhost:8080/)**
* [ ] Admin user created
* [ ] Suggested plugins installed
* [ ] `hello-jenkins` job builds **SUCCESS**
* [ ] Firewall rule in place or port changed if needed

---

