
# xdax-cli Installation Guide

This guide provides step-by-step instructions to set up `xdax`, a script to switch between Apple Silicon (ARM64) and Intel-based Python environments, along with compiling Python from the source.

## Prerequisites

Before proceeding, **remove all existing Python installations**, including any managed by `pyenv`. We will compile Python from the source.

## Step 1: Update `.zshrc`

Add the following script to your `.zshrc` file to configure paths for the Python environments:

```bash
export PATH="$HOME/.python-3.12.2-arm64/bin:$PATH"

xdax() {
    if [[ -z "$1" ]]; then
        echo "Usage: xdax {apple|intel} {command}"
        echo "Usage: xdax activate [name]"
        echo "Usage: xdax help"
        return 1
    fi

    if [[ "$1" == "apple" ]]; then
        python_cmd="$HOME/.python-3.12.2-arm64/bin/python3"
        pip_cmd="$HOME/.python-3.12.2-arm64/bin/pip3"
        brew_cmd='/opt/homebrew/bin/brew'
    elif [[ "$1" == "intel" ]]; then
        python_cmd="$HOME/.python-3.12.2-x84_64/bin/python3"
        pip_cmd="$HOME/.python-3.12.2-x84_64/bin/pip3"
        brew_cmd='arch -x86_64 /usr/local/bin/brew'
    elif [[ "$1" == "help" ]]; then
        echo "Usage: xdax {apple|intel} {command}"
        echo "Usage: xdax activate [name]"
        echo "Commands:"
        echo "  python [command]   Run a Python command."
        echo "  pip [command]      Run a pip command."
        echo "  brew [command]     Run a Homebrew command."
        echo "  venv [name]        Create a virtual environment."
        echo "Examples:"
        echo "  xdax apple python --version"
        echo "  xdax intel pip list"
        echo "  xdax apple brew install <package>"
        echo "  xdax apple venv myenv"
        echo "  xdax activate myenv"
        echo "  xdax intel pip uninstall -y -r <(xdax intel pip freeze)"
        return 0
    else
        command="$1"
        shift
        if [[ "$command" == "activate" ]]; then
            if [[ -z "$1" ]]; then
                echo "Usage: xdax activate [name]"
                return 1
            fi
            if [[ -d "$1" ]]; then
                source "$1/bin/activate"
                echo "Activated virtual environment '$1'."
                return 0
            else
                echo "Error: Virtual environment '$1' does not exist."
                return 1
            fi
        else
            echo "Unknown command. Use 'help' for usage."
            return 1
        fi
    fi

    shift
    case "$1" in
        python)
            if [[ "${@:2}" == "arch" ]]; then
                "$python_cmd" -c "import platform; print(platform.machine())"
            elif [[ "${@:2}" == "file" ]]; then
                file "$python_cmd"
            elif [[ "${@:2}" == "which" ]]; then
                which "$python_cmd"
            else
                "$python_cmd" "${@:2}"
                cmd_status=$?
                if [[ $cmd_status -ne 0 ]]; then
                    echo "Error: Command not found or execution failed."
                fi
                return $cmd_status
            fi
            ;;
        pip)
            "$pip_cmd" "${@:2}"
            cmd_status=$?
            if [[ $cmd_status -ne 0 ]]; then
                echo "Error: Command not found or execution failed."
            fi
            return $cmd_status
            ;;
        brew)
            eval "$brew_cmd" "${@:2}"
            cmd_status=$?
            if [[ $cmd_status -ne 0 ]]; then
                echo "Error: Brew command failed."
            fi
            return $cmd_status
            ;;
        venv)
            if [[ -z "$2" ]]; then
                echo "Usage: xdax {apple|intel} venv [name]"
                return 1
            fi
            "$python_cmd" -m venv "$2"
            cmd_status=$?
            if [[ $cmd_status -ne 0 ]]; then
                echo "Error: Failed to create virtual environment."
            else
                echo "Virtual environment '$2' created successfully."
            fi
            return $cmd_status
            ;;
        *)
            "$python_cmd" "$@"
            cmd_status=$?
            if [[ $cmd_status -ne 0 ]]; then
                echo "Error: Command not found or execution failed."
            fi
            return $cmd_status
            ;;
    esac
}
```

## Step 2: Install Homebrew for Intel

Run the following command to install Homebrew for Intel:

```bash
arch -x86_64 /bin/bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Step 3: Download and Compile Python from Source

Create a temporary directory for the Python source code:

```bash
mkdir temp
cd temp
```

Download the Python 3.12.2 source code:

```bash
curl -O https://www.python.org/ftp/python/3.12.2/Python-3.12.2.tgz
tar -xzf Python-3.12.2.tgz
cd Python-3.12.2
```

### Apple Silicon (ARM64) Build

Run the following commands to compile Python for Apple Silicon:

```bash
xdax apple brew install pkg-config openssl@3 xz gdbm tcl-tk mpdecimal

GDBM_CFLAGS="-I$(xdax apple brew --prefix gdbm)/include" GDBM_LIBS="-L$(xdax apple brew --prefix gdbm)/lib -lgdbm" ./configure --prefix=$HOME/.python-3.12.2-arm64   --enable-optimizations   --enable-shared   --with-openssl="$(xdax apple brew --prefix openssl@3)"

make -j$(sysctl -n hw.ncpu)
make install
```

### Intel (x86_64) Build

Run the following commands to compile Python for Intel:

```bash
xdax intel brew install pkg-config openssl@3 xz gdbm tcl-tk mpdecimal

GDBM_CFLAGS="-I$(xdax intel brew --prefix gdbm)/include" GDBM_LIBS="-L$(xdax intel brew --prefix gdbm)/lib -lgdbm" arch -x86_64 ./configure --prefix=$HOME/.python-3.12.2-x84_64   --enable-optimizations   --enable-shared   --with-openssl="$(xdax intel brew --prefix openssl@3)"

arch -x86_64 make -j$(sysctl -n hw.ncpu)
arch -x86_64 make install
```

### Clean Up

After installation, clean up the temporary files:

```bash
cd ..
rm -rf Python-3.12.2 temp
```

## Usage

To run `xdax`, use the following commands:

- **See xdax help**: 
  ```bash
  xdax help
  ```

- **Create a virtual environment**: 
  ```bash
  xdax intel venv <env_name>
  or
  xdax apple venv <env_name>
  ```

- **Activate a virtual environment**: 
  ```bash
  xdax activate <env_name>
  ```
  
- **Run Python commands**: 
  ```bash
  xdax apple python --version
  xdax intel python script.py
  ```
  
- **Run Pip commands**:
  ```bash
  xdax apple pip install <package>
  xdax intel pip uninstall <package>
  ```

- **Run Brew commands**:
  ```bash
  xdax apple brew install <package>
  xdax intel brew install <package>
  ```