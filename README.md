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
  docker.io/ollama/ollama:latest
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

# Launch the Engine
Using Ollama ROCm image  
--device /dev/kfd & /dev/dri: GPU passthrough (In my case, RX7800XT)  
-v ollama_data: Persistent storage for the models to stop needing to download on restart
HSA_OVERIDE_GFX_VERSION=11.0.0: Forces architecture for RX7800XT used in the ROCm stack
```
podman run -d \
  --name gemma-translator \
  --device /dev/kfd \
  --device /dev/dri \
  -v ollama_data:/root/.ollama \
  -p 11434:11434 \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  docker.io/ollama/ollama:rocm
```

# Pull the Model
I am using Gemma 3
```
podman exec -it gemma-translator ollama pull gemma3:12b
```

# Using Gemma 3 for Translations

```
pip install ollama
```

# Create translate_i18n.py
Change SOURCE_FILE to to the local file path of your intended file to translate  
Change TARGET_LANGS as required
```
import json
import ollama

# --- CONFIG ---
SOURCE_FILE = 'en-US.json'
TARGET_LANGS = ['es-ES', 'fr-FR']
MODEL = 'gemma3:12b'

def translate_text(text, target_lang):
    """Sends a single string to the containerized Gemma 3."""
    if not text or not isinstance(text, str):
        return text

    prompt = f"""Translate the following UI string into {target_lang}.
    - Preserve all placeholders like {{name}}, {{count}}, or %s.
    - Maintain the tone (formal/informal) of the original.
    - Output ONLY the translated text.
    
    English: {text}"""

    try:
        response = ollama.generate(model=MODEL, prompt=prompt)
        return response['response'].strip().strip('"')
    except Exception as e:
        print(f"Error translating '{text}': {e}")
        return text

def recursive_translate(data, target_lang):
    """Walks through the JSON tree recursively."""
    if isinstance(data, dict):
        # If it's a dictionary, translate all its values
        return {k: recursive_translate(v, target_lang) for k, v in data.items()}
    elif isinstance(data, list):
        # If it's a list (e.g., an array of strings), translate each item
        return [recursive_translate(item, target_lang) for item in data]
    elif isinstance(data, str):
        # Base case: it's a string, translate it
        print(f"  Translating: {data[:30]}...")
        return translate_text(data, target_lang)
    else:
        # Return numbers/booleans as they are
        return data

def main():
    with open(SOURCE_FILE, 'r', encoding='utf-8') as f:
        source_data = json.load(f)

    for lang in TARGET_LANGS:
        print(f"\n--- Starting Translation for {lang} ---")
        translated_data = recursive_translate(source_data, lang)
        
        output_file = f"{lang}.json"
        with open(output_file, 'w', encoding='utf-8') as f:
            json.dump(translated_data, f, indent=2, ensure_ascii=False)
        print(f"Finished! Saved to {output_file}")

if __name__ == "__main__":
    main()
```

# If you want to push some of the load onto RAM beyond the 16GB VRAM
```
podman exec -it gemma-translator ollama pull gemma3:27b-it-q4_K_M
```

# Start Podman
```
podman start gemma-translator
```

# Run the .py file
In your konsole
```
cd ./file_location/
python translate_i18n.py
```

```
podman run -d \
  --name ollama-container \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  -e HSA_ENABLE_SDMA=0 \
  -e OLLAMA_NUM_PARALLEL=4 \
  docker.io/ollama/ollama:rocm
```

alt for if it doesn't work
```
podman run -d \
  --name ollama-7800xt \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  -v ollama_7800:/root/.ollama \
  -p 11434:11434 \
  -e HIP_VISIBLE_DEVICES=0 \
  -e HSA_ENABLE_SDMA=0 \
  docker.io/ollama/ollama:rocm
podman run -d \
  --name ollama-6600 \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  -v ollama_6600:/root/.ollama \
  -p 11435:11434 \
  -e HIP_VISIBLE_DEVICES=1 \
  -e HSA_OVERRIDE_GFX_VERSION=10.3.0 \
  -e HSA_ENABLE_SDMA=0 \
  docker.io/ollama/ollama:rocm
```
