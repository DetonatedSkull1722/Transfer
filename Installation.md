Thought for a couple of seconds


**Offline Ubuntu 20.04 LTS: The Complete “First-Timer” Guide**
*You have no Internet on your target machine—and that’s OK! We’ll use a second PC to fetch everything you need, then transfer it via USB/DVD to your offline box.*

---

## 📖 1. Introduction & “Why Offline?”

* **Scenario**: You’ve just installed Ubuntu 20.04 LTS on a machine (the **Offline PC**), but it’s air-gapped—no network.
* **Goal**: Install essential packages (build tools, Python libs), Docker, NVIDIA drivers + CUDA + cuDNN, TensorFlow-GPU, and Node.js, *without* ever connecting this machine to the Internet.
* **Key Idea**: Use a helper tool called **apt-offline** on both machines. On the offline box, generate a “wish list” (a signature file). On the online box, download *exactly* that list of files, bundle them up, then copy back.

---

## 🖥️ 2. Meet Your Two PCs

| Name            | Network Status        | Role                                                   |
| --------------- | --------------------- | ------------------------------------------------------ |
| **Internet PC** | ✅ Has Internet access | Downloads & bundles .deb files (and other installers). |
| **Offline PC**  | ❌ No Internet         | Generates signatures & installs from local bundles.    |

> All file moves happen via USB stick, DVD, or similar “sneakernet” media.

---

## 🔑 3. Core Workflow (Loop)

1. **Offline PC** → *Generate* a signature file (`.sig`) listing everything you want.
2. **Internet PC** → *Download & bundle* all `.deb` files (and index updates) into a ZIP.
3. **Offline PC** → *Install* from that ZIP; it populates `apt`’s cache so you can run normal `apt install` commands.

Repeat this three-step loop for **any** new package or library.

---

## ⚙️ 4. Prerequisites on Both Machines

1. **Ubuntu 20.04 LTS** installed.
2. A USB stick (or DVD) with ≥ 1 GB free.
3. On **both** machines, install `apt-offline` once:

   ```bash
   sudo apt update
   sudo apt install apt-offline
   ```

---

## 🛠️ 5. Step 1: Generate the Signature (Offline PC)

1. **Mount your USB** (assume it auto-mounts at `/media/usb`).
2. **Pick your package(s)**—for example:

   * Essential build tools: `build-essential`
   * Python magic library: `python3-magic`
   * Docker Engine: `docker-ce`
   * Node.js (via Ubuntu repo): `nodejs`
3. **Generate** the signature file (`offline.sig`):

   ```bash
   sudo apt-offline set /media/usb/offline.sig \
       --install-packages build-essential,python3-magic,docker-ce,nodejs
   ```

   * **What happens?**

     * `apt-offline` scans apt’s databases and writes to `offline.sig` exactly which `.deb` files (and versions) are needed to satisfy those packages plus dependencies.
4. **Safely unmount** the USB:

   ```bash
   sudo umount /media/usb
   ```
5. **Physically move** the USB stick to the **Internet PC**.

---

## 🌐 6. Step 2: Download & Bundle (Internet PC)

1. **Insert** the USB (auto-mounts to `/media/$USER/…/usb`).
2. **Ensure apt-offline** is present (you did this in Prerequisites).
3. **Copy over** any extra APT configuration if you use third-party repos (e.g. NodeSource) and their GPG keys—*before* you run the next command.
4. **Run** the downloader:

   ```bash
   cd ~/Downloads
   sudo apt-offline get /media/$USER/usb/offline.sig \
       --bundle ubuntu-offline-packages.zip
   ```

   * This fetches **all** required `.deb` files (plus updated index fragments) and zips them into a single file: `ubuntu-offline-packages.zip`.
5. **Verify** the ZIP:

   ```bash
   ls -lh ubuntu-offline-packages.zip
   ```
6. **Copy** `ubuntu-offline-packages.zip` onto the USB:

   ```bash
   cp ubuntu-offline-packages.zip /media/$USER/usb/
   sudo umount /media/$USER/usb
   ```
7. **Move** the USB back to the **Offline PC**.

---

## 💾 7. Step 3: Install from the Bundle (Offline PC)

1. **Mount** the USB again (`/media/usb`).
2. **Import** the bundle into apt’s cache:

   ```bash
   sudo apt-offline install /media/usb/ubuntu-offline-packages.zip
   ```

   * This places **all** the downloaded `.deb` files into `/var/cache/apt/archives/`.
3. **Install** your packages normally:

   ```bash
   sudo apt update
   sudo apt install build-essential python3-magic docker-ce nodejs
   ```
4. **Verify** each:

   * `gcc --version`
   * `python3 -c "import magic; print(magic.__version__)"`
   * `docker run hello-world`
   * `node --version && npm --version`

> ⭐️ *Tip*: If you ever add another package, just go back to **Step 1** with a new signature (e.g. `apt-offline set /media/usb/new.sig --install-packages nginx`), then repeat **Step 2** and **Step 3**.

---

## 🐳 8. Docker Engine (Offline)

* **On Internet PC**:

  ```bash
  sudo apt-get download docker-ce docker-ce-cli containerd.io
  cp *.deb /media/usb/  # copy these three files
  ```
* **On Offline PC**:

  ```bash
  sudo dpkg -i /media/usb/docker-ce*.deb /media/usb/containerd.io*.deb
  sudo apt-get install -f    # to pull from local cache if needed
  docker run hello-world
  ```

---

## 🚀 9. NVIDIA Driver + CUDA + cuDNN (Offline)

1. **Disable Secure Boot** in BIOS (unsigned modules won’t load).
2. **On Internet PC**, download:

   * NVIDIA driver `.run` installer
   * CUDA Toolkit `.deb` (`cuda-repo-ubuntu2004_*.deb`)
   * cuDNN tarball (`cudnn-*-linux-x64-v7*.tgz`)
     Copy all three to USB.
3. **On Offline PC**:

   ```bash
   sudo apt install gcc-multilib dkms     # if not already present
   sudo sh /media/usb/NVIDIA-Linux-*.run
   sudo dpkg -i /media/usb/cuda-repo-*.deb
   sudo apt update
   sudo apt install cuda
   tar -xzf /media/usb/cudnn-*.tgz
   sudo cp cuda/include/cudnn*.h /usr/local/cuda/include/
   sudo cp cuda/lib64/libcudnn* /usr/local/cuda/lib64/
   sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
   sudo ln -s /usr/local/cuda/lib64/libcudnn.so.* /usr/local/cuda/lib64/libcudnn.so
   ```
4. **Reboot** & **Verify**:

   ```bash
   nvidia-smi
   nvcc --version
   ```

---

## 🧠 10. TensorFlow-GPU 2.3.1 Offline

1. **On Internet PC**:

   ```bash
   python3 -m pip install --upgrade pip
   pip download tensorflow-gpu==2.3.1 -d /media/usb/tfpkgs
   ```
2. **On Offline PC**:

   ```bash
   sudo apt install python3-pip python3-venv
   python3 -m venv ~/tf-env && source ~/tf-env/bin/activate
   pip install --no-index --find-links=/media/usb/tfpkgs tensorflow-gpu==2.3.1
   python - <<EOF
   import tensorflow as tf
   print("GPU available:", tf.test.is_gpu_available())
   EOF
   ```

---

## 📦 11. Node.js: Two Options

### A) Via apt-offline (APT-managed)

1. **Step 1 & 2** exactly as above, but include `nodejs` in `--install-packages`.
2. **Step 3**:

   ```bash
   sudo apt install nodejs
   ```

### B) Standalone Binary (simplest)

1. **On Internet PC**:

   ```bash
   curl -O https://nodejs.org/dist/v16.20.0/node-v16.20.0-linux-x64.tar.xz
   cp node-*.tar.xz /media/usb/
   ```
2. **On Offline PC**:

   ```bash
   sudo mkdir -p /opt/nodejs
   sudo tar -xJf /media/usb/node-*.tar.xz -C /opt/nodejs --strip-components=1
   sudo ln -s /opt/nodejs/bin/{node,npm,npx} /usr/local/bin/
   node --version && npm --version
   ```

---

## ✅ 12. Final Checks & Tips

* **Logs**: If `apt` complains, inspect `/var/log/apt/term.log`.
* **Broken Packages**:

  ```bash
  sudo dpkg --configure -a
  sudo apt-get install -f
  ```
* **Fresh Signatures**: Always re-run **Step 1** when adding new packages.
* **Disk Space**: Bundles can be hundreds of megabytes—use a sufficiently large USB.

---

***Congratulations!*** You now have a fully-featured Ubuntu 20.04 system—compilers, Python libraries, Docker, GPU drivers, TensorFlow, and Node.js—all without ever touching the Internet on your production box.

> *Next step*: Would you like me to package this into a polished PDF for you?
