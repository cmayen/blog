
# Ubuntu, Docker,  and Ollama on AMD to analyze logs.

Creating a new Ubuntu server, with Docker installed, and an Ollama container using LLM to act as a system reporting tool for raising flags on warnings, errors, and critical issues on the machine.

---

My previous daily driver rig wasn't being used anymore and since it had a GPU in it, I decided to experiment. Most AI and LLM installations are running on NVIDIA however this machine is running an old AMD graphics card and an AMD CPU, so there are a few differences that need to be taken care of.  This is an old(ish) machine, built with bargain bin parts in 2019. A lot of heavy games while offloading video encoding to CPU and fumbled loops in code that weren't caught right away. Along with other things... It was a dev box and a game box and a media box all at the same time. It should be interesting to see what kind of report we end up with at the end.


## Read The Friendly Manual
All of this was put together through experimentation and reading the documentation that the packages used provide. They tell us everything we need to know to use them and what the requirements are. The tricky part comes when we work to put them all together and make something different.


## KubeKraft
"Read The Friendly Manual" is something I picked up from Mischa van den Burg in the [KubeCraft](https://www.skool.com/mischa) community. When I first heard that it made me laugh,  I could really appreciate how the different tone of the phrase (compared to what we're used to) actually invites people to look at the manuals and documentation. If it helps people want to learn and succeed, then I love it!
The [KubeCraft](https://www.skool.com/mischa) community is a fantastic group of people with a wide range of skills from absolute beginners to retired veteran experts. It's a great community to both learn and teach at the same time, and not just about tech and Kubernetes. I highly recommend checking it out.


## Prep
Make sure you have backed up everything on the machine we will be installing Ubuntu on, or pick up another physical drive to use and ensure that the only drive hooked up is the one we want to install to. More drives can always be added later.

Get a drink and possibly a sandwich.  This will take a while.  Some of the steps below do not have output for the more simple commands shown. I am assuming enough knowledge to install the minimal server and know how to ssh into it remotely.


## Download the ISO and boot to the Ubuntu Live Environment

[https://ubuntu.com/download](https://ubuntu.com/download)

For this exercise let's use Ubuntu Server 24.04 LTS.
Download the iso file and flash it to a usb drive with `dd` or another program like balena etcher.

Put the drive in the machine and boot to the usb.

Step through the installation process and choose the minimized installation. Include the search for third party drivers. Install the OpenSSH Server and set keys if you like. *Do not include any feature snaps like docker.* We will install everything after we have a working base system on this chunk of metal.


## Boot to the fresh install and take care of first steps.

Do your updates!
```shell
sudo apt update -y && sudo apt upgrade -y
```

We need a firewall!
```shell
sudo apt install ufw -y
```

Open the port for the OpenSSH server.
```shell
sudo ufw allow 22
```

Enable the firewall to turn it on and start at boot.
```shell
sudo ufw enable
```

Reboot the system, then check we have deny all rules, and an opening for port 22.
```shell
sudo ufw status verbose
```

Get the IP address of the machine so we can ssh into it from our primary workstation.
```shell
ip a
```


## Installing Docker and necessary AMD packages.

First let's update some group memberships, and allow the use of hardware.
```shell
sudo usermod -aG render,video $LOGNAME
```

Retrieve the docker and jq packages. It would be a good idea to have an editor like vim, also ensure wget is installed, and gpg will be needed for keys. One call sounds good for all of them.
```shell
sudo apt install -y docker.io jq vim wget gpg
```

Get the AMD GPU installer package from the radeon repo and install it. (At the time of this document, the version was 6.3.4)
```shell
wget https://repo.radeon.com/amdgpu-install/6.3.4/ubuntu/jammy/amdgpu-install_6.3.60304-1_all.deb
sudo apt install -y ./amdgpu-install_6.3.60304-1_all.deb
sudo amdgpu-install --usecase=dkms
```

The amd-container-toolkit is required to get everything working so the containers can talk to the GPU. The package is not available by the default repositories, so we need to add the information.

First get the keys:
```shell
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null
```

Now add the repo information to the sources:
```shell
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/amd-container-toolkit/apt/ noble main" | sudo tee /etc/apt/sources.list.d/amd-container-toolkit.list
```

In order for apt to know the repo exists, we need to update it. We may as well check for updates/upgrades too.
```shell
sudo apt update -y && sudo apt upgrade -y
```

Install the amd-container-toolkit and docker-compose-v2
```shell
sudo apt install amd-container-toolkit docker-compose-v2
```

Configure the runtime, and restart docker.
```shell
sudo amd-ctk runtime configure
sudo systemctl restart docker
```

Test that the GPU is available using rocm/rocm-terminal 
```shell
sudo docker run --runtime=amd -e AMD_VISIBLE_DEVICES=all rocm/rocm-terminal amd-smi monitor
```

You should see GPUs listed, in my case, there is only one.
```output
GPU  POWER   GPU_T   MEM_T   GFX_CLK   GFX%   MEM%   ENC%   DEC%      VRAM_USAGE
  0   10 W   47 °C   46 °C   800 MHz    0 %    0 %    N/A    N/A    0.0/  8.0 GB
```

Looks good, let's get a list of the docker containers so we can remove the one we just ran. Good housekeeping and all.
```shell
sudo docker ps -a
```

```output
CONTAINER ID   IMAGE                COMMAND               CREATED          STATUS                      PORTS                                           NAMES
30c6e6bfbfe2   rocm/rocm-terminal   "amd-smi monitor"     2 minutes ago   Exited (0) 2 minutes ago                                                   boring_cartwright
```

```shell
sudo docker remove 30c6e6bfbfe2
```


## Installing Ollama as a docker container.
Since this is an AMD based machine, we will be using ollama:rocm 

Create the docker-compose.yml file for the ollama service. Note that in this example I am completely opening cors with OLLAMA_ORIGINS=*  This generally is a bad idea and should be commented out, or at least defined to only allow from certain domains. I am leaving cors wide open as a personal convenience for interacting with it remotely. This machine is behind two routers with firewalls already, so I'm not worried about cors security right now.

docker-compose.yml
```yaml
services:
  ollama:
    image: ollama/ollama:rocm
    restart: unless-stopped
    container_name: ollama
    devices:
      - "/dev/kfd:/dev/kfd"
      - "/dev/dri:/dev/dri"
    network_mode: "bridge"
    ports:
      - "11434:11434"
    environment:
      - "OLLAMA_HOST=0.0.0.0:11434"
      - "OLLAMA_ORIGINS=*"
    volumes:
      - ollama_data:/root/.ollama
volumes:
  ollama_data:
```

Bring up the services via docker-compose
```shell
sudo docker compose up -d
```

Get the list of docker containers so we can get the id for adding the model.
```shell
sudo docker ps -a
```

```output
CONTAINER ID   IMAGE                COMMAND               CREATED         STATUS                     PORTS                                           NAMES
739a8ed9c057   ollama/ollama:rocm   "/bin/ollama serve"   3 minutes ago     Up 3 minutes                 0.0.0.0:11434->11434/tcp, :::11434->11434/tcp   ollama
```

Now that we have the container id, we'll use that to get our model pulled for ollama. It looks like gemma3 might be a good choice for this experiment. Feel free to use and add any models you like that your GPU can handle.
```shell
sudo docker exec -it 739a8ed9c057 ollama pull gemma3
```

Test it! Make a curl request and generate a response with the model chosen, and setting stream to false.
```shell
curl http://localhost:11434/api/generate -d '{ "model": "gemma3", "stream": false, "prompt": "Hi Gemma!"}'
```

The response will be in json format, with a response message similar to the following:
```output
"Hi there! It's lovely to hear from you. 😊 How’s your day going so far?

Is there anything you’d like to chat about, or were you just saying hello?"
```


## Locating system issues in the logs:
A shell script will be used to go through system logs and the journal to look for errors, warnings and other bits of critical information we should know about from the previous day. The output will be via echo so directing the output can be controlled.

log-issue-search-24h.sh
```sh
#!/bin/sh

DATE_S=$(date --date="yesterday"  +"%Y-%m-%d")
DATE_Y=$(date --date="yesterday"  +"%Y-%m-%d 00:00:00")
DATE_T=$(date --date="today"  +"%Y-%m-%d 00:00:00")

echo "========================================================="
echo " Log Data - Errors, Warnings, and Criticals - $DATE_S"
echo "========================================================="

# Scan the /var/log folder for *.log files and loop them looking for the 
# different error, warning, and critical log entries.
LOGFILES=$(find /var/log -name "*.log" -mtime -1)
for LOGFILE in $LOGFILES; do
  # echo -e "\n=== LOGFILE: $LOGFILE ==="
  grep -EiwH "warning|error|critical|alert|fatal" $LOGFILE | grep "$DATE_S"
done

# Call to journalctl requesting critical information from yesterday.
# echo -e "\n === LOGFILE: journalctl ==="
journalctl -S "$DATE_Y" -U "$DATE_T" --no-pager --priority=3..0
```

Make the file executable
```shell
chmod +x log-issue-search-24h.sh
```


## Creating the pre-prompt:
It will not be enough to just pass all this log data to the LLM by itself. We need to tell the AI what it's role is, and what it should do with the data given. Create a pre-prompt text file.

pre-prompt.txt
```wrap
You are a devops system administrator in charge of monitoring logs for issues and suggesting resolutions. Go through all of the following log information, generate a detailed report about the issues found, and include suggestions for resolutions of the issues.
Do not explain what a MCE or it's components are. Do not explain what each log file is for. Provide a summary of issues and stay focused on explaining the those issues with examples of resolutions.
---
```


## Setting up to call the ollama API and parse the response.
In order to call the ollama API passing the pre-prompt and log data, and then parsing the response to get the core message, we'll use python instead of shell.

ollama-logs-report.py
```python

import requests
import json
import sys


# load pre-prompt.txt
try:
  with open("pre-prompt.txt","r") as file:
    prePromptContent = file.read()
except FileNotFoundError:
  print(f"Error: File not found: 'pre-prompt.txt'")
  sys.exit(1)  # Exit with failure


# load passed in filename
filePath = sys.argv[1]
try:
    with open(filePath, "r") as file:
        fileContent = file.read()
except FileNotFoundError:
    print(f"Error: The file '{filePath}' was not found.")
    sys.exit(1)  # Exit with failure
except Exception as e:
    print(f"An error occurred: {e}")
    sys.exit(1)  # Exit with failure


# append the content to the pre-prompt
promptContent = prePromptContent + "\n" + fileContent


# setup the api data package
url = "http://localhost:11434/api/generate"  # Replace with your API endpoint
payload = {
    "model": "gemma3",
    "stream": False,
    "prompt":promptContent
}


# call the api
try:
    response = requests.post(url, json=payload)
    response.raise_for_status()  # Raise an exception for HTTP errors (4xx or 5xx)
except requests.exceptions.RequestException as e:
    print(f"Error during POST request: {e}")
    sys.exit(1)  # Exit with failure

if response.status_code == 200:
    try:
        # Parse the JSON response
        response_data = response.json()

        # dig out the response
        if "response" in response_data:
            print(response_data["response"])
            sys.exit(0)  # Exit with success
        else:
            print("Unexpected response format:", response_data)
            sys.exit(1)  # Exit with failure
    except json.JSONDecodeError:
        print("Response is not valid JSON.")
        print(f"Raw response content: {response.text}")
        sys.exit(1)  # Exit with failure
else:
    print(f"POST request failed with status code: {response.status_code}")
    print(f"Response content: {response.text}")
    sys.exit(1)  # Exit with failure
```


## Here we go! First test.
```shell
sudo ./log-issue-search-24h.sh > log-issue-results.txt
python3 ollama-logs-report.py log-issue-results.txt > ollama-logs-report.txt
```

This may take a few moments.


## Look at the generated AI system report.
Open up the ollama-logs-report.txt
```shell
vim ollama-logs-report.txt
```

```wrap


Okay, let's break down these logs and understand what's happening. These logs from a system named "fusion" show a recurring pattern of machine check errors, along with various other issues.

**Key Observations and Interpretation:**

1. **Recurring Machine Check Errors:**  The most concerning aspect is the repeated "Machine Check" errors. These indicate hardware problems – specifically, issues with the CPU's internal memory and error-correcting capabilities.  These errors are extremely bad and strongly suggest a failing CPU.

   * **`mce: [Hardware Error]: CPU ... Machine Check: 0 ...`**: This confirms a machine check, the error-detection system within the CPU is reporting an error.  The `Bank 5: bea0000000000108` part provides a specific memory address that triggered the error.

2. **`udev-worker` Failure:**  The `udev-worker` process is consistently failing to call `EVIOCSKEYCODE`. This error indicates a problem with device detection or handling by the udev system (which manages device drivers). It’s likely related to the underlying hardware problems causing the CPU errors.

3. **TSC (Time Stamp Counter) Issues:** The logs also show the TSC (Time Stamp Counter) being reported incorrectly.  This is a common symptom of failing CPUs.  The TSC is used for accurate timekeeping, and a failing CPU will often generate incorrect TSC values.

4. **Microcode Version:** The logs contain the microcode version (8701034), which is the firmware version of the CPU.  While updating microcode *might* temporarily mask the underlying problem, it will not fix a fundamentally failing CPU.

**What This Means (Diagnosis):**

* **CPU Failure:** The primary cause is almost certainly a failing CPU. The machine check errors, TSC issues, and the fact that the system continues to boot and function (albeit poorly) despite these errors suggest a hardware problem.
* **Memory Issues:** The error address (`bea0000000000108`) points to a specific memory location, which is consistent with a failing CPU. CPUs have internal memory caches and error correction, and when those fail, you'll see these machine check errors.
* **Potential for Further Damage:**  Continuing to run the system with a failing CPU will almost certainly lead to further data corruption and system instability.

**Recommendations:**

1. **Immediate Action: Shutdown:** The system *must* be shut down immediately to prevent further data corruption.  Do *not* attempt to continue using it.

2. **Hardware Diagnosis:** Get the system professionally diagnosed by a qualified computer repair technician. They'll need to perform thorough testing to confirm the CPU failure.

3. **CPU Replacement:** The solution is to replace the CPU.  This is a complex process and should ideally be done by a professional to avoid damaging other components.

4. **Data Backup (if possible):** If there's any chance to quickly recover data, do so *before* shutting down.  However, given the potential for corruption, this may not be a priority.
```

This machine has some problems I need to look at. The AI is giving a worst case scenario based on the information it found. This could be a bios update needed or bad drivers.  It could also be a burning out CPU. Honestly, I have really beat this machine up over the years. There are a lot of hours of 90% load or higher. This is also looking at the logs during the initial post-installation process when not all the amd drivers have been loaded yet.

I'll let this run a few days and keep an eye on it.


## Make tweaks to the pre-prompt
The AI may put extra information in the report you don't care about. Things like explaining what each log file does and what MCEs are. If you find the output teaches you things, it's worth keeping it all. Otherwise, update the pre-prompt file with more detailed instructions about what to include, and what to not ramble on about.


## What now?
We have a report file. We also have a combined issue log file. One thing we can do is create an email to send to the administrator with the report as the body, and the combined logs as an attachment. This could fill an inbox quickly, and possibly expose vulnerabilities to outside networks. So I won't be doing that.

One thing I want to do is have a web interface or API for a cluster dashboard.  Setup a docker container that will act as a receiving point where multiple servers can send the log-issue-search-24h.sh output to. That receiving server would then run the logs through the LLM generating multiple reports, then do another AI run to summarize all of the reports into one. Obviously, automate it all!

That sounds like a fun next step, cool deal! For now I will likely make a cron job and manually look at the report when I log in until I can get more infrastructure going and make a tighter network stack. Maybe setup some Kubernetes pods and have them use the reporter server too.
