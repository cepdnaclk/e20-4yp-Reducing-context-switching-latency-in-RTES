# RISC-V 32-bit Performance Testing Environment Setup Guide

This guide provides step-by-step instructions to set up a Docker-based RISC-V 32-bit environment for performance testing in a custom directory structure.

---

## 📁 01. Setup Local Environment

### 1.1 Required Folder Structure
```
working_dir/                 # For this project `/projects/Reducing_ctxswl_in_RTES/code`
└── FYP_Docker/              # Contains Docker assets (provided to you)
    ├── readme.md            # Original documentation
    └── riscv_env.tar        # Pre-built Docker image archive
```

### 1.2 Create Performance Testing Directory
```bash
# Create the custom environment directory in the working directory. (`/projects/Reducing_ctxswl_in_RTES/code`)
mkdir -p ./Performance_Test

# Navigate to the new directory
cd ./Performance_Testing
```

### 1.3 Copy Docker Assets
```bash
# Copy the provided FYP_Docker directory to the Performance Testing directory.
cp /path/to/working_dir/FYP_Docker .

# Navigate to the docker archive directory
cd FYP_Docker

# Verify files exist
ls -lh
# Should show:
#   riscv_env.tar
#   readme.md
```

### 1.4 Load Docker Image from Archive
```bash
# Load the image (restores original name/tag)
docker load -i riscv_env.tar

# Verify successful load
docker images | grep riscv
# Expected output:
# riscv-32bit-env   v1    <IMAGE_ID>   ...   <SIZE>
```

### 1.5 Create Workspace Directory for Mounting
```bash
# Come back to the Performance Testing directory
cd ..

# Verify the directory
pwd      # Expected output: /projects/Reducing_ctxswl_in_RTES/code/Performance_Test

# Create directory for persistent experiments
mkdir -p Experiments
```

### 1.6 Final directory structure
```
Performance_Testing/
├── Experiments/          # Your persistent workspace (mounted to /FYP)
├── FYP_Docker/           # Contains Docker assets
|    ├── readme.md        # Documentation for Docker
|    └── riscv_env.tar    # Docker image archive
└── README.md             # Documentation for performance testing
```

### 1.7 Run the Container
```bash
# Start a tempoprary container with workspace mounted to /FYP
docker run -it --rm --name RFP_PMT \
  -v $(pwd)/Experiments:/FYP \
  riscv-32bit-env:v1
```

> 💡 **Inside container tips**:
> - Your host's `Experiments/` directory is accessible at `/FYP`
> - All changes in `/FYP` persist after container stops
> - Press `Ctrl+P` then `Ctrl+Q` to detach without stopping container
> - When `exit` from the temporary container (`--rm`), it will auto delete the container and free the name to use

### 1.8 Customize & Commit New Image
After making changes inside the container (e.g., installing perf tools):

```bash
# EXIT the container first (type 'exit' or press Ctrl+D)
exit

# Commit container state to new image
docker commit RFP_PMT riscv-32bit-env:v1-Performance_Test

# Verify new image exists
docker images | grep Performance_Test
# Should show:
# riscv-32bit-env   v1-Performance_Test   <NEW_IMAGE_ID>   ...   <SIZE>
```

> **📝 Note:**
> - To see all the _running containers_ use the command `docker ps`
> - To see all the containers _(running and stopped)_ use the command `docker ps -a`
> - To see all the existing docker images use the command `docker images | grep Performance_Test`

### 1.9 Verify Installation with Test Program
Start new container from customized image
```bash
docker run -it --rm --name RFP_PMT \
  -v $(pwd)/Experiments:/FYP \
  riscv-32bit-env:v1-Performance_Test
```

Inside container, run verification tests:
```bash
cd /FYP
mkdir /test && cd /test
echo '#include <stdio.h>
int main() {
    printf("Success: 32-bit RISC-V Environment is working!\n");
    return 0;
}' > test.c
```

Compile and run
```bash
# Compile using the 32-bit GCC:
riscv32-unknown-elf-gcc -o test test.c

# Run using our new alias:
run32 test

# Expected output: 
# Success: 32-bit RISC-V Environment is working!
```

### 1.10 Exit Environment
```bash
# Inside container: type 'exit' to stop and exit
exit

# Container stops automatically (--rm flag not used in initial run)
# Your work in /FYP is safely stored in host's Experiments/ directory
```

---

## 🐳 1.11 Essential Docker Commands Cheat Sheet

| Task | Command | Description |
|------|---------|-------------|
| **List running containers** | `docker ps` | Show active containers |
| **List all containers** | `docker ps -a` | Show running + stopped containers |
| **List images** | `docker images` | Show all local images |
| **Stop container** | `docker stop RFP_PMT` | Graceful shutdown |
| **Force stop** | `docker kill RFP_PMT` | Immediate termination |
| **Remove container** | `docker rm RFP_PMT` | Delete stopped container |
| **Remove image** | `docker rmi riscv-32bit-env:v1` | Delete unused image |
| **View logs** | `docker logs RFP_PMT` | See container output after exit |
| **Re-enter container** | `docker start -ai RFP_PMT` | Restart and attach to existing container |
| **Clean unused resources** | `docker system prune -a` | ⚠️ Removes ALL stopped containers, unused images, networks |

---

## 🔒 Best Practices & Warnings

1. **Never store project files inside container**  
   → Always use mounted volumes (`Experiments/` → `/FYP`)

2. **Image management**  
   ```bash
   # Before deleting images, check dependencies:
   docker ps -a --filter "ancestor=riscv-32bit-env:v1"
   ```

3. **Avoid sudo with Docker** (security risk)  
   ```bash
   # Fix permissions once (then logout/login):
   sudo usermod -aG docker $USER
   ```

4. **Backup your work**  
   ```bash
   # Export customized image for sharing/backup:
   docker save -o riscv_perf_env.tar riscv-32bit-env:v1-Performance_Test
   ```

5. **Disk space monitoring**  
   ```bash
   docker system df  # Show Docker disk usage
   ```

---

> 💡 **Need help?**  
> Check Docker logs: `docker logs RFP_PMT`  
> Inspect container state: `docker inspect RFP_PMT`  
> Get shell in stopped container: `docker start -ai RFP_PMT`

---

*Document version: 1.0 • Updated: February 2026*  
*Environment: RISC-V 32-bit GNU Toolchain • Ubuntu 22.04 Base*