# QTC OCR

uv venv ocr_env

source ocr_env/bin/activate

uv pip install pymupdf ocrmypdf opencv-python-headless pillow numpy odfpy


pdfdetach -saveall 1920-tal.pdf   



### Try 1 - no preprocessing
```
import os
import subprocess
import logging
from pathlib import Path
import fitz  # PyMuPDF

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

def process_pdf(pdf_path, output_dir, languages='swe+eng'):
    try:
        base_name = Path(pdf_path).stem
        issue_dir = Path(output_dir) / base_name
        issue_dir.mkdir(parents=True, exist_ok=True)

        with fitz.open(pdf_path) as doc:
            total_pages = len(doc)
            
            for page_num in range(total_pages):
                try:
                    # Create page-specific PDF
                    single_page_pdf = issue_dir / f"{base_name}_page_{page_num+1}.pdf"
                    with fitz.open() as single_doc:
                        single_doc.insert_pdf(doc, from_page=page_num, to_page=page_num)
                        single_doc.save(single_page_pdf)
                    
                    # OCR with ocrmypdf
                    ocr_pdf = issue_dir / f"{base_name}_ocr_page_{page_num+1}.pdf"
                    subprocess.run([
                        'ocrmypdf',
                        '-l', languages,
                        '--force-ocr',
                        '--optimize', '3',
                        str(single_page_pdf),
                        str(ocr_pdf)
                    ], check=True)
                    
                    # Extract text
                    text_file = issue_dir / f"{base_name}_page_{page_num+1}.txt"
                    with fitz.open(ocr_pdf) as ocr_doc:
                        text = ocr_doc[0].get_text()
                        text_file.write_text(text, encoding='utf-8')
                    
                    # Cleanup
                    single_page_pdf.unlink()
                    ocr_pdf.unlink()
                    
                    logging.info(f"Processed page {page_num+1}/{total_pages} of {pdf_path.name}")

                except Exception as page_error:
                    logging.error(f"Error processing page {page_num+1} of {pdf_path.name}: {page_error}")
                    continue

    except Exception as doc_error:
        logging.error(f"Error processing document {pdf_path.name}: {doc_error}")

def process_pdfs(input_dir, output_dir):
    input_path = Path(input_dir)
    for pdf_file in input_path.glob('*.pdf'):
        process_pdf(pdf_file, output_dir)

# Usage
if __name__ == "__main__":
    process_pdfs('input_pdfs', 'output_text')

```

### Try 2 - with OpenCV cleaning and preprocessing

```
import os
import subprocess
import logging
from pathlib import Path
import cv2
import numpy as np
import fitz  # PyMuPDF
from PIL import Image

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def preprocess_image(image):
    # Convert to grayscale
    gray = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)
    
    # Noise reduction
    denoised = cv2.fastNlMeansDenoising(gray, h=10)
    
    # Adaptive thresholding
    thresh = cv2.adaptiveThreshold(denoised, 255, 
        cv2.ADAPTIVE_THRESH_GAUSSIAN_C, 
        cv2.THRESH_BINARY, 21, 10)
    
    # Morphological operations to clean text
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (2,2))
    cleaned = cv2.morphologyEx(thresh, cv2.MORPH_CLOSE, kernel)
    
    return cleaned

def process_pdf(pdf_path, output_dir, languages='swe+eng'):
    try:
        base_name = Path(pdf_path).stem
        issue_dir = Path(output_dir) / base_name
        issue_dir.mkdir(parents=True, exist_ok=True)

        with fitz.open(pdf_path) as doc:
            total_pages = len(doc)
            
            for page_num in range(total_pages):
                try:
                    # Extract page with original DPI metadata
                    page = doc.load_page(page_num)
                    pix = page.get_pixmap()
                    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
                    
                    # Calculate DPI from PDF dimensions (1 Point = 1/72 inch)
                    pdf_width_pt = page.rect.width
                    pdf_height_pt = page.rect.height
                    dpi_x = pix.width / (pdf_width_pt/72)
                    dpi_y = pix.height / (pdf_height_pt/72)
                    avg_dpi = int((dpi_x + dpi_y) / 2)

                    # Preprocess image
                    preprocessed = preprocess_image(np.array(img))
                    
                    # Save preprocessed image with DPI metadata using Pillow
                    preprocessed_pil = Image.fromarray(preprocessed)
                    preprocessed_path = issue_dir / f"{base_name}_preprocessed_page_{page_num+1}.png"
                    preprocessed_pil.save(str(preprocessed_path), dpi=(avg_dpi, avg_dpi))

                    # OCR with explicit DPI parameter
                    ocr_pdf = issue_dir / f"{base_name}_ocr_page_{page_num+1}.pdf"
                    subprocess.run([
                        'ocrmypdf',
                        '-l', languages,
                        '--image-dpi', str(avg_dpi),  # Explicit DPI setting
                        '--force-ocr',
                        '--optimize', '3',
                        '--oversample', '300',  # Force minimum 300 DPI processing
                        '--skip-big', '50',  # Skip images larger than 50 megapixels
                        str(preprocessed_path),
                        str(ocr_pdf)
                    ], check=True)
                    
                    # Extract text
                    text_file = issue_dir / f"{base_name}_page_{page_num+1}.txt"
                    with fitz.open(ocr_pdf) as ocr_doc:
                        text = ocr_doc[0].get_text()
                        text_file.write_text(text, encoding='utf-8')
                    
                    # Cleanup
                    preprocessed_path.unlink()
                    ocr_pdf.unlink()
                    
                    logging.info(f"Processed page {page_num+1}/{total_pages} of {pdf_path.name} (DPI: {avg_dpi})")

                except Exception as page_error:
                    logging.error(f"Error processing page {page_num+1} of {pdf_path.name}: {page_error}")
                    continue

    except Exception as doc_error:
        logging.error(f"Error processing document {pdf_path.name}: {doc_error}")

def process_pdfs(input_dir, output_dir):
    input_path = Path(input_dir)
    for pdf_file in input_path.glob('*.pdf'):
        process_pdf(pdf_file, output_dir)

# Usage
if __name__ == "__main__":
    process_pdfs('input_pdfs', 'output_text')

```



https://community.element14.com/members-area/personalblogs/b/blog/posts/cleaning-old-scanned-book-pages-with-linux-bash-commands

### Install poppler (pdftotext etc)
```
sudo apt-get install poppler-utils
```

### Unpack decade portfolios
```
pdfdetach -saveall 1920-tal.pdf
pdfdetach -saveall 1930-tal.pdf
...
...
```

### Install ocrmypdf   
```
apt install ocrmypdf
```

### Install Swedish language pack
```
sudo apt-get install tesseract-ocr-swe
```

### Make smaller PDFs
Message from ocrmypdf:  
```
The output file size is 6.16× larger than the input file.                                                                                                         _validation.py:351
Possible reasons for this include:                                                                                                                                                  
--force-ocr was issued, causing transcoding.                                                                                                                                        
The optional dependency 'jbig2' was not found, so some image optimizations could not be attempted.                                                                                  
PDF/A conversion was enabled. (Try `--output-type pdf`.)    
```

#### Build JBIG2 encoder (make smaller PDFs)
https://ocrmypdf.readthedocs.io/en/latest/jbig2.html

```
[sudo] apt install autotools-dev automake libtool libleptonica-dev
```
```
git clone https://github.com/agl/jbig2enc
cd jbig2enc
./autogen.sh
./configure && make
[sudo] make install
```
### Run OCR and extract text

```
#!/bin/bash

# Create a directory for the output
mkdir -p qtc_ocr_output

# Loop through all QTC issue files in the current directory
for pdf in *.pdf; do
    # Get the base name (issue name) of the PDF file (without extension)
    base_name=$(basename "$pdf" .pdf)
    
    # Create a directory for this issue's output
    mkdir -p "qtc_ocr_output/$base_name"
    
    # Get the number of pages in the issue
    num_pages=$(pdfinfo "$pdf" | grep Pages | awk '{print $2}')
    
    # Process each page separately
    for ((page=1; page<=num_pages; page++)); do
        # Extract single page from issue PDF
        pdfseparate -f $page -l $page "$pdf" "qtc_ocr_output/$base_name/page_${page}.pdf"
        
        # OCR the single page, use Swedish and English, remove old 
        ocrmypdf -l swe+eng --force-ocr "qtc_ocr_output/$base_name/page_${page}.pdf" "qtc_ocr_output/$base_name/ocr_page_${page}.pdf"
        
        # Extract and save text from page
        pdftotext -layout "qtc_ocr_output/$base_name/ocr_page_${page}.pdf" "qtc_ocr_output/$base_name/page_${page}.txt"
        
        # Remove temporary PDF files
        rm "qtc_ocr_output/$base_name/page_${page}.pdf" "qtc_ocr_output/$base_name/ocr_page_${page}.pdf"
    done
done
```

