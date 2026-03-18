# Dual GPU Podman ollama container using Vulkan to bypass the RDNA2/RDNA3 incompatibility with ollama ROCM
```
podman run -d \
  --name ollama-vulkan \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  -e OLLAMA_VULKAN=1 \
  -e OLLAMA_DEBUG=1 \
  -e GGML_VK_VISIBLE_DEVICES=1,2 \
  docker.io/ollama/ollama:latest
```
# Host a web interface in a separate podman container to be more pleasant to interact with the model
```
podman run -d \
  --name open-webui \
  -p 3000:8080 \
  --add-host=host.containers.internal:host-gateway \
  -v open-webui-data:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.containers.internal:11434 \
  ghcr.io/open-webui/open-webui:main
```
# Pull a model for local storage so you don't download it every time. I'm using glm-4.7-flash for 24gb VRAM (16gb 7800xt, 8gb 6600)
```
podman exec -it ollama-vulkan pull glm-47-flash
```
# Open the web interface at http://localhost:3000

# Use Roo Code to run the local model in VS Code for code assist
- Download the Roo Code extension in VS Code
- Navigate to roo in the sidebar
- Select settings (gear icon)
- Select ollama for API provider
- Base URL: http://localhost:11434
- Model ID: (in my case) glm-4.7-flash
- Context Window: 8192 or 16384 (upper limit for 24GB VRAM so you don't out of memory)

.rooignore for VS Code workspace
```
# .rooignore - Prevent Roo Code from blowing up LLM context

# Version Control
.git/

# Dependencies & Virtual Environments
node_modules/
vendor/
venv/
.venv/
env/
.env

# Build Outputs & Compiled Binaries
dist/
build/
out/
target/
bin/
obj/
*.class
*.dll
*.exe
*.so
*.o

# Large Media & Assets
*.png
*.jpg
*.jpeg
*.gif
*.ico
*.mp4
*.mp3
*.wav
*.svg

# Large Data & Log Files
*.log
*.sql
*.sqlite
*.db
*.csv
*.tsv
*.parquet
*.jsonl

# Lockfiles (Huge context hogs, terrible for reasoning)
package-lock.json
yarn.lock
pnpm-lock.yaml
poetry.lock
Pipfile.lock
Cargo.lock

# Minified files
*.min.js
*.min.css
*.bundle.js

# OS specific
.DS_Store
Thumbs.db
```

# Containerized-LLM-Fedora-KDE-
Using a Fedora KDE system, establish a containerized instance of an LLM. In this case, I am using mine for i18n translations with Google's Gemma 3

Using: Podman, Ollama,

# Prepare the Host
Grant device permissions to talk to the GPU  
Enable direct access to hardware for the containers  
```
sudo usermod -a -G video,render $USER
sudo setsebool -P container_use_devices true
```

# Initiate a local model into a new repo
## Use repomix to condense the repo into one small file

## Prompt 1
```
We are starting work in a new repository. Please use your tools to list the files in the root directory. Then, read ONLY the README.md and the primary dependency file (e.g., package.json, Cargo.toml, or requirements.txt). Based strictly on those files, give me a brief 3-bullet-point summary of what this project does and the core tech stack being used. Do not read any other code files yet.
```

## Prompt 2
```
I want to implement a new feature related to [insert feature, e.g., user authentication]. Please use your list_files or search tools to find the top 3-5 files most likely related to this feature. List the file paths for me, but do not use the read_file tool yet.
```

## Prompt 3
```
Read the files you just identified. Explain how [Feature] is currently handled in these specific files, and write out a step-by-step plan for how we should modify them to achieve [New Goal]. Wait for my approval before making any edits.
```

## Prompt 4
