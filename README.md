# Acer Swift 16 AI Linux Setup

This repository contains configuration tweaks, hardware quirk overrides, and setup instructions for running Linux (Fedora 44 Workstation) on the **Acer Swift 16 AI**.

---

## 1. Touchpad Pressure Pad Fix (libinput)

Fixes touchpad sensitivity and physical click recognition for the PixArt PIXA4813 touchpad:

```bash
sudo mkdir -p /etc/libinput
sudo cp local-overrides.quirks /etc/libinput/local-overrides.quirks
```

Restart libinput/GDM or reboot to apply:
```bash
sudo systemctl restart gdm
```

---

## 2. Facial Recognition Setup (Howdy)

Howdy provides Windows Hello™-style facial authentication using the built-in infrared (IR) camera (`/dev/video2`). Because Fedora lacks pre-packaged `dlib` binaries and standard Copr repos do not support newer Fedora releases (such as F44), Howdy and dlib must be built from source inside a Python virtual environment.

### Step 1: Install Build Dependencies & Development Headers

```bash
sudo dnf install -y \
  python3 python3-pip python3-setuptools python3-wheel \
  meson ninja-build cmake gcc gcc-c++ \
  pam-devel inih-devel libevdev-devel \
  python3-opencv opencv-devel python3-devel \
  libX11-devel openblas-devel lapack-devel
```

### Step 2: Configure Dynamic Linker for `/usr/local/lib64`

Ensure libraries installed into `/usr/local/lib64` are resolvable:

```bash
echo "/usr/local/lib64" | sudo tee /etc/ld.so.conf.d/local.conf
sudo ldconfig
```

### Step 3: Clone Howdy and Set Up Python Virtual Environment

Clone Howdy and install the required Python packages (`dlib`, `numpy`, `opencv-python`) into an isolated virtual environment:

```bash
git clone https://github.com/boltgolt/howdy.git ~/howdy
cd ~/howdy

python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install dlib numpy opencv-python
```

### Step 4: Build and Install Howdy with Meson

Point Meson to the virtualenv's Python binary so `pam_howdy.so` and `/usr/local/bin/howdy` invoke this environment:

```bash
meson setup build -Dpython_path=$(pwd)/.venv/bin/python3
meson compile -C build
sudo meson install -C build
```

### Step 5: Symlink PAM Module for Fedora

Fedora looks for PAM modules in `/usr/lib64/security/`, but Meson installs `pam_howdy.so` into `/usr/local/lib64/security/`:

```bash
sudo mkdir -p /usr/lib64/security
sudo ln -sf /usr/local/lib64/security/pam_howdy.so /usr/lib64/security/pam_howdy.so
```

### Step 6: Download Dlib Facial Models

Download the pre-trained neural network models:

```bash
cd /usr/local/share/dlib-data
sudo ./install.sh
```

### Step 7: Configure Howdy Settings

Edit the configuration file (`sudo howdy config` or edit `/usr/local/etc/howdy/config.ini`):

```ini
[core]
no_confirmation = true

[video]
# The IR camera on Acer Swift 16 AI is /dev/video2
device_path = /dev/video2

# Increase dark threshold for better contrast in IR frames
dark_threshold = 80
```

> **Note:** Verify the camera stream with `ffplay /dev/video2` or `sudo howdy test`.

### Step 8: Configure PAM Integration

Add `auth sufficient pam_howdy.so` to the PAM configuration files you want to authenticate with Howdy.

#### 1. `sudo` (`/etc/pam.d/sudo`)
Insert right after `#%PAM-1.0` (before `system-auth`):
```text
auth        sufficient    pam_howdy.so
```

#### 2. GDM Login (`/etc/pam.d/gdm-password`)
Insert right after `pam_selinux_permit.so` (line 2):
```text
auth        sufficient    pam_howdy.so
```

#### 3. GNOME Lock Screen (`/etc/pam.d/gnome-lockscreen`)
Add to the very top (line 1):
```text
auth        sufficient    pam_howdy.so
```

#### 4. Polkit (`/etc/pam.d/polkit-1`)
If `/etc/pam.d/polkit-1` does not exist, copy it from `/usr/lib/pam.d/polkit-1` first:
```bash
sudo cp /usr/lib/pam.d/polkit-1 /etc/pam.d/polkit-1
```
Then add before `system-auth`:
```text
auth        sufficient    pam_howdy.so
```

### Step 9: SELinux Configuration

Because PAM and GDM run in confined SELinux domains (`xdm_t`), SELinux blocks them from accessing Python binaries and libraries located in home directories.

To ensure Howdy works reliably across GDM and lock screen:
Edit `/etc/selinux/config` and set:
```ini
SELINUX=disabled
```
*(or `SELINUX=permissive`)*, then reboot to apply.

### Step 10: Enroll Face Models & Verify

Enroll your face:
```bash
sudo howdy add
```

Test camera and recognition:
```bash
sudo howdy test
```

Verify `sudo` authentication in a new terminal window:
```bash
sudo echo "Howdy authentication working!"
```
