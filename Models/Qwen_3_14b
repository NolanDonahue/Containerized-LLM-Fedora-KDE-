# Quick Deploy after it's installed
```
 podman run -d \
  --name qwen-translator \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  -e HSA_ENABLE_SDMA=0 \
  -e OLLAMA_NUM_PARALLEL=4 \
  docker.io/ollama/ollama:rocm
python translate_i18n.py 
```
# Pull the Model
Qwen 3 14b-q4_K_M
Set it to think (for more complex writing such as idioms) or not think
```
podman exec -it qwen-translator ollama pull qwen3:14b
ollama run qwen3:14b --think=false
```

# Create translate_i18n.py
Change SOURCE_FILE to to the local file path of your intended file to translate  
Change TARGET_LANGS as required
```
import json
import ollama

# --- CONFIG ---
SOURCE_FILE = '/home/ndonahue/Desktop/Code/LLM/Qwen/en-US.json'
TARGET_LANGS = ['nl-NL', 'pt-BR', 'pl-PL', 'ru-RU', 'de-DE']
MODEL = 'qwen3:14b-q4_K_M'
CHUNK_SIZE = 50  # How many lines to translate at once

def translate_json_chunk(json_chunk, target_lang):
    """Sends a block of JSON to the model to translate all at once."""
    prompt = (
        f"Translate the values in this JSON object into {target_lang}. "
        "Keep the exact same JSON structure. DO NOT translate the keys. "
        "Preserve placeholders like {{name}} or %s. "
        "Output ONLY valid JSON, starting with { and ending with }."
    )
    
    messages = [
        {'role': 'system', 'content': prompt},
        {'role': 'user', 'content': json.dumps(json_chunk, indent=2)}
    ]

    try:
        response = ollama.chat(
            model=MODEL, 
            messages=messages,
            options={'num_gpu': 99, 'temperature': 0.1} # Low temp keeps it strict
        )
        
        result_text = response['message']['content'].strip()
        
        # Strip out markdown code blocks if the model adds them
        if result_text.startswith("```json"):
            result_text = result_text[7:]
        if result_text.endswith("```"):
            result_text = result_text[:-3]
            
        return json.loads(result_text.strip())
        
    except json.JSONDecodeError:
        print("\n  [!] Model returned invalid JSON. Falling back to original chunk.")
        return json_chunk
    except Exception as e:
        print(f"\n  [!] Error: {e}")
        return json_chunk

def get_flat_paths(data, current_path=None):
    """Flattens the JSON into a list of (path, text) tuples."""
    if current_path is None:
        current_path = []
    
    paths = []
    if isinstance(data, dict):
        for k, v in data.items():
            paths.extend(get_flat_paths(v, current_path + [k]))
    elif isinstance(data, list):
        for i, v in enumerate(data):
            paths.extend(get_flat_paths(v, current_path + [i]))
    elif isinstance(data, str):
        paths.append((current_path, data))
        
    return paths

def main():
    with open(SOURCE_FILE, 'r', encoding='utf-8') as f:
        source_data = json.load(f)

    # Flatten the tree to grab all strings easily
    all_strings = get_flat_paths(source_data)
    
    for lang in TARGET_LANGS:
        print(f"\n==========================================")
        print(f"--- Translating {lang} in chunks of {CHUNK_SIZE} ---")
        print(f"==========================================")
        
        # Build a fresh copy to inject translations into
        translated_data = json.loads(json.dumps(source_data)) 
        
        # Process in chunks
        for i in range(0, len(all_strings), CHUNK_SIZE):
            chunk_paths = all_strings[i:i + CHUNK_SIZE]
            
            # Create a temporary dictionary for the LLM
            temp_dict = { "::".join(map(str, path)): text for path, text in chunk_paths }
            
            print(f"  Sending items {i+1} to {min(i+CHUNK_SIZE, len(all_strings))}")
            translated_chunk = translate_json_chunk(temp_dict, lang)
            
            # Map the translated values back to their exact nested locations
            for path_str, translated_text in translated_chunk.items():
                path = path_str.split("::")
                target = translated_data
                for key in path[:-1]:
                    # Handle array indices vs dict keys
                    key = int(key) if key.isdigit() else key
                    target = target[key]
                final_key = int(path[-1]) if path[-1].isdigit() else path[-1]
                target[final_key] = translated_text
        
        output_file = f"{lang}.json"
        with open(output_file, 'w', encoding='utf-8') as f:
            json.dump(translated_data, f, indent=2, ensure_ascii=False)
        print(f"\nFinished! Saved to {output_file}")

if __name__ == "__main__":
    main()
```
