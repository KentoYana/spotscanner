# SpotScanner

SpotScanner is a command-line tool for quantitative analysis of fungal **spot-test plates**.  
It detects a printed marker on the plate, segments the image into a fixed grid, and measures colony growth in each spot, outputting results as CSV files and diagnostic images.

The tool is designed for use in **fungal genetics** experiments (e.g., *Neurospora* spot tests) with either multi-channel replicators or manual pipetting.

---

## Features

- Supports two plate layouts:
  - **Replicator mode**: 8 × 12 matrix
  - **Pipette mode**: 4 × 6 matrix
- Uses a **printed marker** to align and normalize images.
- Automatically:
  - Detects the marker on the plate
  - Segments the plate into spots
  - Quantifies colony growth per spot
- Outputs:
  - `*_converted.png` images for visual QC
  - `*_results.csv` files with per-spot measurements
- Provides helper commands to:
  - View a text manual (man-style)
  - Open or regenerate the marker
  - Generate a BibTeX entry for citation

---

## Requirements

- **Python**: 3.8+ (tested with 3.12)
- Python libraries:
  - `opencv-python` (or `opencv-python-headless`)
  - `numpy`
  - Standard library modules: `argparse`, `csv`, `datetime`, `glob`, `os`, `shutil`, `subprocess`, `sys`, `pathlib`

Install the Python dependencies, for example:

```bash
pip install numpy opencv-python
```

---

## Installation

### Directory layout

SpotScanner expects the following directory structure under a single **install root**:

```text
INSTALL_ROOT/
  bin/
    spotscanner          # small wrapper script (optional)

  spotscanner/
    __init__.py
    __main__.py
    ... (other Python modules if needed)

  marker/
    Marker1.png
    Marker2.png
    ...
    Marker16.png

  template/
    marker.pdf           # Printable marker
    marker.tex           # LaTeX source for the marker
    spotscanner.1        # Text manual in man(1) style
    (optional) PDF documentation
```

Here, `INSTALL_ROOT` is the directory that contains the `spotscanner`, `marker`, and `template` subdirectories.

### How SpotScanner finds its install root

At runtime, SpotScanner determines `INSTALL_ROOT` as:

1. If the environment variable **`SPOTSCANNER_ROOT`** (or the legacy `SPOTSCANNER`) is set:  
   use that as the install root.
2. Otherwise:  
   infer the root as the parent of the `spotscanner` package directory, based on `__main__.py`’s location.

So you have two typical ways to install:

#### Option A: Keep everything together and run with `python -m`

1. Place the directory tree like this:

   ```text
   /path/to/spotscanner-install/
     spotscanner/
     marker/
     template/
   ```

2. From `/path/to/spotscanner-install`, run:

   ```bash
   python -m spotscanner --help
   ```

   This works because Python can import the local `spotscanner` package and `__main__.py` can find `INSTALL_ROOT` from its own path.

#### Option B: Use an environment variable + a shell wrapper

1. Place the directory tree anywhere, e.g.:

   ```text
   /opt/spotscanner/
     spotscanner/
     marker/
     template/
   ```

2. Set the environment variable `SPOTSCANNER_ROOT` to that directory, e.g. in your shell config:

   ```bash
   export SPOTSCANNER_ROOT=/opt/spotscanner
   ```

3. Create a small wrapper script named `spotscanner` somewhere in your `PATH` (e.g. `~/bin/spotscanner`):

   ```bash
   #!/usr/bin/env bash
   python -m spotscanner "$@"
   ```

   Make it executable:

   ```bash
   chmod +x ~/bin/spotscanner
   ```

4. Now you can simply run:

   ```bash
   spotscanner --help
   ```

---

## Command-line usage

### Basic syntax

```bash
spotscanner [-v] [-e] [-r]
            [-o {manual,marker-pdf,marker-tex,citation}] \
            -a {pipette,replicator}
            [-s SINGLE | -m]
            [-t 1..10]
```

### Options

#### General options

- `-v`, `--version`  
  Show the version information and the citation, then exit.

- `-e`, `--example`  
  Show short usage examples and exit.

- `-r`, `--readme`  
  View the SpotScanner manual (`template/spotscanner.1`) with `less`.  
  Equivalent to:

  ```bash
  spotscanner -r
  ```

  which internally runs `less` on the installed manual file.

- `-o`, `--output {manual,marker-pdf,marker-tex,citation}`  
  Open or output auxiliary resources:

  - `manual`  
    Print the text manual (man-style) to standard output.  
    This is useful for piping, e.g.:

    ```bash
    spotscanner -o manual | less
    ```

  - `marker-pdf`  
    Open the printable marker PDF (`template/marker.pdf`) with the system viewer.

  - `marker-tex`  
    - Print the **LaTeX source** of the marker to the terminal.
    - Write it to `marker.tex` in the **current working directory**.
    - Prints a hint:

      ```text
      You can run `lualatex marker.tex` in this directory to generate marker.pdf.
      ```

  - `citation`  
    Print a BibTeX entry to the terminal and write it to `spotscanner.bib` in the **current working directory**, e.g.:

    ```bibtex
    @Manual{spotscanner,
      title  = {SpotScanner: A tool to recognize and analyze images of spot-test plates in fungal genetics},
      author = {Kento Yanagisawa},
      year   = {2025},
      note   = {version X.Y.Z},
      url    = {Input Later},
    }
    ```

#### Analysis mode options

- `-a`, `--analyze {pipette,replicator}`  
  Choose the plate layout. This option is **required** to perform image analysis.

  - `pipette`  
    4 × 6 matrix (manual pipetting layout)

  - `replicator`  
    8 × 12 matrix (multi-channel replicator layout)

#### Image selection options (mutually exclusive)

- `-s`, `--single SINGLE`  
  Analyze **only the specified image file**.  
  `SINGLE` should be a `.jpg` / `.JPG` / `.jpeg` file in the current directory.

- `-m`, `--multi`  
  Analyze **all images** in the current directory with extension `.jpg``, `.JPG`, or `.jpeg`.

You must choose **either** `--single` or `--multi` when analyzing images.

#### Marker recognition threshold

- `-t`, `--threshold N`  
  Threshold for marker recognition, where `N` is an integer from `1` to `10`.

  - Default: `7` (if `-t` is omitted)
  - Internally interpreted as `N / 10`, i.e. a value between `0.1` and `1.0`.

Higher values generally make marker detection stricter.

---

## Typical workflows

### 1. Generate and print the marker

To obtain the marker PDF directly:

```bash
spotscanner -o marker-pdf
```

This will open `marker.pdf`, which you can print and place under your spot-test plate.

If you want to regenerate the marker from LaTeX:

```bash
spotscanner -o marker-tex
lualatex marker.tex
```

This produces `marker.pdf` in the current directory.

---

### 2. Analyze a single image (pipette mode)

1. Place the printed marker under the spot-test plate.
2. Take a photograph of the plate including the entire marker.
3. Copy the image (e.g. `plate1.jpg`) into a working directory.
4. Run:

   ```bash
   spotscanner -a pipette -s plate1.jpg
   ```

This will:

- Detect the marker and align the image
- Segment it into a 4 × 6 grid
- Output:
  - `plate1_converted.png` — processed image for QC
  - `plate1_results.csv` — CSV with columns such as:

    ```text
    Time, Image, Threshold, row, column, Colony
    ```

---

### 3. Analyze all images in a directory (replicator mode)

1. Put all your plate images (e.g. `plate1.jpg`, `plate2.jpg`, …) into one directory.
2. Run:

   ```bash
   spotscanner -a replicator -m
   ```

This will:

- Process each `*.jpg` / `*.JPG` / `*.jpeg` file in the directory
- For each image, create a `*_converted.png` and `*_results.csv`
- Create two subdirectories:

  - `converted_YYYY-MM-DD_HH-MM-SS/` — contains all `*_converted.png` images
  - `results_YYYY-MM-DD_HH-MM-SS/` — contains all `*_results.csv` files

The timestamp is based on the start time of the analysis run.

---

### 4. Getting help and examples

- Show help:

  ```bash
  spotscanner --help
  ```

- Show short usage examples:

  ```bash
  spotscanner --example
  ```

- View the manual with `less`:

  ```bash
  spotscanner --readme
  ```

---

## Citation

SpotScanner can show its own version and citation:

```bash
spotscanner --version
```

The citation is also available as a BibTeX entry via:

```bash
spotscanner -o citation
```

which writes `spotscanner.bib` in the current directory.

## License

This project is licensed under the MIT License.
