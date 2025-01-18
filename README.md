# GeoMetadataEditor

## Overview
`GeoMetadataEditor` is a Python project designed for embedding geospatial and camera-related metadata into image files. It reads metadata from a CSV file, processes it, and encodes it into the image's EXIF data. After that, it allows you to view the EXIF metadata embedded in the image file, confirming the success of the embedding process.

The project is based on two key components:
1. **Embedding metadata into image** files by reading from a CSV.
2. **Viewing metadata** (EXIF data) embedded into the output image.

This README details the required setup, how to write metadata into images, and how to view it.

---

## Table of Contents
1. [Installation](#installation)
2. [Usage](#usage)
    1. [Read Metadata from CSV](#read-metadata-from-csv)
    2. [Embed Metadata into Image](#embed-metadata-into-image)
    3. [View EXIF Metadata](#view-exif-metadata)
3. [Folder Structure](#folder-structure)
4. [How to Run](#how-to-run)
5. [Troubleshooting](#troubleshooting)

---

## 1. Installation

This project requires **Python 3.x** and the following Python packages:

1. **Pillow** - For image processing.
2. **piexif** - For working with EXIF metadata.

To install the necessary packages, run:

```bash
pip install pillow piexif
```

---

## 2. Usage

### 2.1 Read Metadata from CSV

The project reads a CSV file containing the metadata for an image. The CSV file should contain key-value pairs of metadata information.

**CSV Format Example (`input/metadata.csv`):**

```csv
Make,Canon
Model,EOS 70D
Software,Adobe Photoshop Lightroom
FocalLength,50
ApertureValue,2.8
ExposureProgram,0
MeteringMode,5
ISOSpeedRatings,400
WhiteBalance,0
SceneCaptureType,0
lat_ref,N
long_ref,E
altitude,150.0
latitude,34.0522
longitude,-118.2437
timestamp,2025-01-01T12:00:00
```

### 2.2 Embed Metadata into Image

After reading metadata from the CSV, the project processes this metadata and embeds it into the image EXIF data.

**Key Steps:**
1. Open the image file.
2. If the image has no existing EXIF data, it generates a new EXIF data structure.
3. Metadata is added under the following sections:
   - **Camera Information:** Make, Model, Software, etc.
   - **GPS Information:** Latitude, Longitude, Altitude, Timestamp.
4. The updated EXIF data is written back into the image and saved to the specified output location.

### Code to Embed Metadata:
```python
from PIL import Image
import piexif
import csv
import os

# Directory for storing files
os.chdir("g:\\Programming_Playground\\Python_Programming\\GeoMetadataEditor")

def read_metadata(csv_path):
    metadata = {}
    try:
        with open(csv_path, mode='r') as file:
            reader = csv.reader(file)
            for row in reader:
                if row:
                    metadata[row[0]] = row[1]
    except Exception as e:
        print(f"Error reading CSV: {e}")
    return metadata

def encode_metadata(image_path, output_path, geo_data, metadata, model, date_time, latitude, longitude):
    try:
        print(f"Opening image at path: {image_path}")
        # Open image
        image = Image.open(image_path)
        # Process and clean EXIF data
        image.info["exif"] = b""
        exif_data = image.info.get("exif", b"")
        
        # If no EXIF data, create new EXIF metadata
        if not exif_data:
            print(f"Warning: No EXIF data found in the image '{image_path}'. Generating new EXIF data.")
            exif_data = piexif.dump({})
        exif_dict = piexif.load(exif_data)

        # Add Camera Data to EXIF
        exif_dict["0th"] = {
            piexif.ImageIFD.Make: metadata.get("Make", "").encode(),
            piexif.ImageIFD.Model: model.encode(),
            piexif.ImageIFD.XResolution: (72, 1),
            piexif.ImageIFD.YResolution: (72, 1),
            piexif.ImageIFD.ResolutionUnit: 2,
            piexif.ImageIFD.Software: metadata.get("Software", "").encode(),
            piexif.ImageIFD.Orientation: 1,
        }

        # Add GPS Data (Latitude, Longitude, Altitude)
        gps_ifd = {
            piexif.GPSIFD.GPSLatitudeRef: geo_data["lat_ref"].encode(),
            piexif.GPSIFD.GPSLatitude: convert_to_dms(latitude),
            piexif.GPSIFD.GPSLongitudeRef: geo_data["long_ref"].encode(),
            piexif.GPSIFD.GPSLongitude: convert_to_dms(longitude),
            piexif.GPSIFD.GPSAltitude: (int(geo_data.get("altitude", 0) * 100), 100),
            piexif.GPSIFD.GPSAltitudeRef: 0,
            piexif.GPSIFD.GPSDateStamp: date_time.encode(),
        }
        exif_dict["GPS"] = gps_ifd

        # Additional Date & Time EXIF Data
        exif_dict["Exif"] = {
            piexif.ExifIFD.DateTimeOriginal: date_time.encode(),
            piexif.ExifIFD.DateTimeDigitized: date_time.encode(),
        }

        # Embed EXIF and Save
        exif_bytes = piexif.dump(exif_dict)
        image.save(output_path, "jpeg", exif=exif_bytes)
        print(f"Metadata successfully encoded and saved to {output_path}")
    except Exception as e:
        print(f"Error embedding metadata: {e}")

def convert_to_dms(value):
    """Convert decimal degrees to DMS for GPS metadata."""
    degrees = int(value)
    minutes = int((value - degrees) * 60)
    seconds = round((value - degrees - minutes / 60) * 3600, 5)
    return ((degrees, 1), (minutes, 1), (int(seconds * 100), 100))

```

### 2.3 View EXIF Metadata

Once the metadata is embedded into the image, you can view the EXIF data in the output image. This is done by extracting the EXIF data and displaying its key-value pairs.

Here is the code to read and display the EXIF data of the output image:

```python
def view_exif_metadata(image_path):
    try:
        # Open the image to extract EXIF data
        image = Image.open(image_path)
        exif_data = image._getexif()
        
        if exif_data:
            print(f"\nEXIF Metadata for image: {image_path}")
            for tag, value in exif_data.items():
                tag_name = piexif.TAGS.get(tag, tag)
                print(f"{tag_name}: {value}")
        else:
            print("\nNo EXIF data found in the image.")
    except Exception as e:
        print(f"Error opening image: {e}")

# View the metadata of the processed image
output_image_path = "output/final_image_with_metadata.jpg"
view_exif_metadata(output_image_path)
```

### 2.4 Helper Functions

- **`convert_to_dms(value)`**: Converts latitude or longitude values from decimal degrees to Degrees, Minutes, and Seconds (DMS), which is required for EXIF GPS metadata format.

---

## 3. Folder Structure

Here’s how the folder structure should look like:

```
GeoMetadataEditor/
    ├── input/
    │   ├── metadata.csv
    │   └── input_image.jpeg
    ├── output/
    │   └── final_image_with_metadata.jpg
    ├── GeoMetadataEditor.ipynb
    └── README.md
```

- **`input/`**: Contains the raw image (`input_image.jpeg`) and the metadata CSV file (`metadata.csv`).
- **`output/`**: Contains the final image (`final_image_with_metadata.jpg`) with the embedded EXIF metadata.
- **`GeoMetadataEditor.ipynb`**: The main script to read metadata, encode it, and view it.
- **`README.md`**: Documentation file.

---

## 4. How to Run

### Step 1: Prepare Files
Place your image file (`input_image.jpeg`) and CSV (`metadata.csv`) in the `input` folder.

### Step 2: Run the Script
In your terminal or command prompt, navigate to the project directory and execute:

```bash
GeoMetadataEditor.ipynb
```

This will process the image, embed the metadata, save the new image to the `output/` folder, and display the EXIF metadata.

### Step 3: View Metadata
To check the EXIF metadata of the processed image (`final_image_with_metadata.jpg`), you can rerun the part of the script dedicated to **viewing metadata**.

---

## 5. Troubleshooting

- **Error reading CSV**: Check the CSV file format and ensure it has no syntax errors.
- **No EXIF data found**: If the input image has no EXIF metadata initially, the script will automatically generate it.
- **Saving error**: Ensure you have write permissions in the output directory.
- **Missing keys in CSV**: Ensure the CSV contains all required keys for the metadata.

--- 


## 📜 **License & Acknowledgments**

This repository is licensed under the **MIT License**. You can use, modify, and distribute it under this license.

- **Author**: S. Pratap Yadav
  - **GitHub**: [iSPYadav01](https://github.com/iSPYadav01)
  - **Portfolio**: [S. Pratap Yadav Portfolio](https://ispyadav01.github.io/Portfolio/)

Follow me on:
- [LinkedIn](https://www.linkedin.com/in/iSPYadav01)
- [Twitter](https://twitter.com/iSPYadav01)

© 2025 Data Dynasty Lab. All Rights Reserved.
```