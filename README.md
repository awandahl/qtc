# QTC OCR


https://www.perplexity.ai/search/how-can-i-run-pdftotext-on-a-p-ui5jRLHsSZGrnELgkg23xg#45

sudo apt install libhunspell-dev   
sudo apt-get install hunspell-sv

uv pip install pip
uv pip install odfpy pymupdf ocrmypdf opencv-python-headless pillow numpy   
uv pip install hunspell spacy python-Levenshtein   
python -m spacy download sv_core_news_sm   





```
from collections import Counter
import re
import os
import logging
from pathlib import Path
import hunspell
import spacy

# Load Swedish NLP model
try:
    nlp = spacy.load("sv_core_news_sm")
except OSError:
    raise RuntimeError("Swedish model required. Run: python -m spacy download sv_core_news_sm")

# Configuration
CONFIG = {
    'input_dir': 'output_text',
    'min_word_length': 4,
    'min_word_freq': 3,
    'custom_dict': 'qtc_custom.dic',
    'base_dict': '/usr/share/hunspell/sv_SE',
    'ocr_prefix': 'QTC_',  # Match your OCR file pattern
    'original_prefix': 'original_'
}

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

def collect_text_files():
    """Collect OCR-processed text files from directory structure"""
    ocr_files = []
    for txt_path in Path(CONFIG['input_dir']).rglob('*.txt'):
        if (txt_path.name.startswith(CONFIG['ocr_prefix']) and 
            not txt_path.name.startswith(CONFIG['original_prefix']) and
            'comparison' not in txt_path.name):
            ocr_files.append(txt_path)
    return ocr_files

def generate_frequency_list(files):
    """Generate word frequency list from OCR texts"""
    word_counter = Counter()
    for file in files:
        with open(file, 'r', encoding='utf-8') as f:
            text = f.read().lower()
            words = re.findall(r'\b[\w+]+\b', text)
            word_counter.update(words)
    return word_counter

def build_custom_dictionary(word_freq):
    """Build custom dictionary using frequency analysis"""
    hobj = hunspell.HunSpell(CONFIG['base_dict'] + '.dic', CONFIG['base_dict'] + '.aff')
    
    known_words = set()
    potential_errors = {}
    
    for word, count in word_freq.items():
        if hobj.spell(word) or len(word) < CONFIG['min_word_length']:
            known_words.add(word)
        else:
            potential_errors[word] = count
    
    # Create custom dictionary
    with open(CONFIG['custom_dict'], 'w', encoding='utf-8') as f:
        f.write("# QTC Amateur Radio Dictionary\n")
        f.write("QTC\nQRG\nQTH\n")  # Add known radio terms
        
        # Add high-frequency potential terms
        for word in sorted(potential_errors, key=lambda x: potential_errors[x], reverse=True):
            if potential_errors[word] >= CONFIG['min_word_freq']:
                f.write(f"{word}\n")
    
    return known_words

def correct_ocr_files(files, known_words):
    """Perform spellcheck and correction on OCR files"""
    hobj = hunspell.HunSpell(CONFIG['base_dict'] + '.dic', CONFIG['base_dict'] + '.aff')
    
    for file in files:
        try:
            with open(file, 'r', encoding='utf-8') as f:
                original_text = f.read()
            
            doc = nlp(original_text)
            corrections = {}
            
            for token in doc:
                if (len(token.text) >= CONFIG['min_word_length'] and
                    token.text.lower() not in known_words and
                    not token.is_punct and
                    not token.like_num):
                    
                    suggestions = hobj.suggest(token.text)
                    if suggestions:
                        corrections[token.text] = suggestions[0]
            
            # Apply corrections
            corrected_text = original_text
            for wrong, right in corrections.items():
                corrected_text = re.sub(r'\b' + re.escape(wrong) + r'\b', right, corrected_text)
            
            # Save corrected file
            corrected_path = file.parent / f"corrected_{file.name}"
            with open(corrected_path, 'w', encoding='utf-8') as f:
                f.write(corrected_text)
            
            logging.info(f"Processed {file.name} with {len(corrections)} corrections")
            
        except Exception as e:
            logging.error(f"Error processing {file.name}: {str(e)[:200]}")

def process_files():
    """Main processing workflow"""
    logging.info("1/4 Collecting text files...")
    ocr_files = collect_text_files()
    
    logging.info("2/4 Generating frequency list...")
    word_freq = generate_frequency_list(ocr_files)
    
    logging.info("3/4 Building custom dictionary...")
    known_words = build_custom_dictionary(word_freq)
    
    logging.info("4/4 Performing corrections...")
    correct_ocr_files(ocr_files, known_words)

if __name__ == "__main__":
    process_files()

```





uv venv ocr_env

source ocr_env/bin/activate

uv pip install pymupdf ocrmypdf opencv-python-headless pillow numpy odfpy
uv pip install odfpy pymupdf ocrmypdf opencv-python-headless pillow numpy


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
import fitz  # PyMuPDF
from odf.opendocument import OpenDocumentText
from odf.style import Style, TextProperties, ParagraphProperties, TableColumnProperties
from odf.table import Table, TableColumn, TableRow, TableCell
from odf.text import P
import cv2
import numpy as np
from PIL import Image

# Configuration
CONFIG = {
    'ENABLE_PREPROCESSING': False,
    'ENABLE_GRAYSCALE': True,
    'ENABLE_DENOISE': True,
    'ENABLE_THRESHOLDING': True,
    'ENABLE_MORPHOLOGY': True,
    'THRESH_BLOCK_SIZE': 21,
    'THRESH_C': 10,
    'LANGUAGES': 'swe+eng',
    'OPTIMIZE_LEVEL': 3,
    'FORCE_OCR': True,
    'SAVE_PREPROCESSED_IMAGES': False,
    'TRIAL_MODE': True,
    'TESSERACT_TIMEOUT': 120,
    'COMPARISON_DOC': True
}

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def preprocess_image(image):
    """Apply preprocessing steps based on configuration"""
    if not CONFIG['ENABLE_PREPROCESSING']:
        return image
    
    if CONFIG['ENABLE_GRAYSCALE']:
        image = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)
    
    if CONFIG['ENABLE_DENOISE']:
        image = cv2.fastNlMeansDenoising(image, h=10)
    
    if CONFIG['ENABLE_THRESHOLDING']:
        image = cv2.adaptiveThreshold(
            image, 255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY,
            CONFIG['THRESH_BLOCK_SIZE'],
            CONFIG['THRESH_C']
        )
    
    if CONFIG['ENABLE_MORPHOLOGY']:
        kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (2,2))
        image = cv2.morphologyEx(image, cv2.MORPH_CLOSE, kernel)
    
    return image

def create_comparison_doc(issue_dir, base_name):
    """Create ODT document with original vs new OCR text"""
    doc = OpenDocumentText()
    
    # Create page break style
    page_break_style = Style(name="PageBreak", family="paragraph")
    page_break_style.addElement(ParagraphProperties(breakbefore="page"))
    doc.automaticstyles.addElement(page_break_style)
    
    # Create header style
    h1style = Style(name="Heading1", family="paragraph")
    h1style.addElement(TextProperties(fontsize="14pt", fontweight="bold"))
    doc.styles.addElement(h1style)
    
    # Add header
    header = P(stylename=h1style, text=f"Comparison for issue: {base_name}")
    doc.text.addElement(header)
    
    # Create table style
    table_style = Style(name="ComparisonTable", family="table")
    col_style = Style(name="TableColumn", family="table-column")
    col_props = TableColumnProperties(columnwidth="8cm")
    col_style.addElement(col_props)
    doc.automaticstyles.addElement(col_style)
    
    table = Table(stylename=table_style)
    table.addElement(TableColumn(numbercolumnsrepeated=2, stylename=col_style))
    
    # Add pages in order
    page_files = sorted(issue_dir.glob(f"{base_name}_page_*.txt"))
    original_files = sorted(issue_dir.glob(f"original_{base_name}_page_*.txt"))
    
    for i, (orig_file, new_file) in enumerate(zip(original_files, page_files)):
        if i > 0:
            doc.text.addElement(P(stylename=page_break_style))
        
        try:
            with open(orig_file, 'r', encoding='utf-8') as f:
                original_text = f.read()
            with open(new_file, 'r', encoding='utf-8') as f:
                new_text = f.read()
            
            row = TableRow()
            
            # Original text cell
            cell = TableCell()
            p = P(text=original_text)
            cell.addElement(p)
            row.addElement(cell)
            
            # New OCR cell
            cell = TableCell()
            p = P(text=new_text)
            cell.addElement(p)
            row.addElement(cell)
            
            table.addElement(row)
        except Exception as e:
            logging.error(f"Error creating comparison row: {str(e)[:200]}")
    
    doc.text.addElement(table)
    output_path = issue_dir / f"{base_name}_comparison.odt"
    doc.save(str(output_path))
    return output_path

def process_pdf(pdf_path, output_dir):
    try:
        base_name = Path(pdf_path).stem
        issue_dir = Path(output_dir) / base_name
        issue_dir.mkdir(parents=True, exist_ok=True)

        original_texts = []
        
        with fitz.open(pdf_path) as doc:
            total_pages = len(doc)
            if CONFIG['TRIAL_MODE']:
                total_pages = min(1, total_pages)
            
            # First pass: Extract original text
            for page_num in range(total_pages):
                try:
                    page = doc.load_page(page_num)
                    original_text = page.get_text()
                    original_path = issue_dir / f"original_{base_name}_page_{page_num+1}.txt"
                    original_path.write_text(original_text, encoding='utf-8')
                    original_texts.append(original_text)
                except Exception as e:
                    logging.error(f"Error extracting original text page {page_num+1}: {str(e)[:200]}")
                    original_texts.append("")

            # Second pass: Process with OCR
            for page_num in range(total_pages):
                try:
                    page = doc.load_page(page_num)
                    pix = page.get_pixmap()
                    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
                    
                    # Calculate DPI
                    pdf_width_pt = page.rect.width
                    pdf_height_pt = page.rect.height
                    dpi_x = pix.width / (pdf_width_pt/72)
                    dpi_y = pix.height / (pdf_height_pt/72)
                    avg_dpi = int((dpi_x + dpi_y) / 2)

                    # Preprocess image
                    preprocessed = preprocess_image(np.array(img))
                    
                    # Build OCR command
                    cmd = [
                        'ocrmypdf',
                        '-l', CONFIG['LANGUAGES'],
                        '--force-ocr',
                        '--optimize', str(CONFIG['OPTIMIZE_LEVEL']),
                        '--tesseract-timeout', str(CONFIG['TESSERACT_TIMEOUT']),
                        '--image-dpi', str(avg_dpi),
                        str(pdf_path),
                        str(issue_dir / "temp_ocr_output.pdf")
                    ]
                    
                    subprocess.run(cmd, check=True, timeout=CONFIG['TESSERACT_TIMEOUT'])
                    
                    # Extract new OCR text
                    with fitz.open(issue_dir / "temp_ocr_output.pdf") as ocr_doc:
                        new_text = ocr_doc.load_page(page_num).get_text()
                        new_text_path = issue_dir / f"{base_name}_page_{page_num+1}.txt"
                        new_text_path.write_text(new_text, encoding='utf-8')

                    logging.info(f"Processed page {page_num+1}/{total_pages}")

                except Exception as page_error:
                    logging.error(f"Error processing page {page_num+1}: {str(page_error)[:200]}")
                    continue

            # Create comparison document
            if CONFIG['COMPARISON_DOC']:
                comparison_path = create_comparison_doc(issue_dir, base_name)
                logging.info(f"Created comparison document: {comparison_path}")

            # Final cleanup
            (issue_dir / "temp_ocr_output.pdf").unlink(missing_ok=True)

    except Exception as doc_error:
        logging.error(f"Error processing document: {str(doc_error)[:200]}")

def process_pdfs(input_dir, output_dir):
    input_path = Path(input_dir)
    for pdf_file in input_path.glob('*.pdf'):
        process_pdf(pdf_file, output_dir)

if __name__ == "__main__":
    logging.info("Starting OCR processing with comparison feature")
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


```
import os
import subprocess
import logging
from pathlib import Path
import fitz  # PyMuPDF
from odf.opendocument import OpenDocumentText
from odf.style import Style, TableColumnProperties
from odf.table import Table, TableColumn, TableRow, TableCell
from odf.text import P
import cv2
import numpy as np
from PIL import Image

# Configuration
CONFIG = {
    'ENABLE_PREPROCESSING': True,
    'ENABLE_GRAYSCALE': True,
    'ENABLE_DENOISE': True,
    'ENABLE_THRESHOLDING': True,
    'ENABLE_MORPHOLOGY': True,
    'THRESH_BLOCK_SIZE': 21,
    'THRESH_C': 10,
    'LANGUAGES': 'swe+eng',
    'OPTIMIZE_LEVEL': 3,
    'FORCE_OCR': True,
    'SAVE_PREPROCESSED_IMAGES': False,
    'TRIAL_MODE': False,
    'TESSERACT_TIMEOUT': 120,
    'COMPARISON_DOC': True
}

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def preprocess_image(image):
    """Apply preprocessing steps based on configuration"""
    if not CONFIG['ENABLE_PREPROCESSING']:
        return image
    
    if CONFIG['ENABLE_GRAYSCALE']:
        image = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)
    
    if CONFIG['ENABLE_DENOISE']:
        image = cv2.fastNlMeansDenoising(image, h=10)
    
    if CONFIG['ENABLE_THRESHOLDING']:
        image = cv2.adaptiveThreshold(
            image, 255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY,
            CONFIG['THRESH_BLOCK_SIZE'],
            CONFIG['THRESH_C']
        )
    
    if CONFIG['ENABLE_MORPHOLOGY']:
        kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (2,2))
        image = cv2.morphologyEx(image, cv2.MORPH_CLOSE, kernel)
    
    return image

def create_comparison_doc(issue_dir, base_name):
    """Create ODT document with original vs new OCR text"""
    doc = OpenDocumentText()
    
    # Create table style
    table_style = Style(name="ComparisonTable", family="table")
    col_style = Style(name="TableColumn", family="table-column")
    col_props = TableColumnProperties(columnwidth="8cm")
    col_style.addElement(col_props)
    doc.automaticstyles.addElement(col_style)
    
    table = Table(stylename=table_style)
    table.addElement(TableColumn(numbercolumnsrepeated=2, stylename=col_style))
    
    # Add pages in order
    page_files = sorted(issue_dir.glob(f"{base_name}_page_*.txt"))
    original_files = sorted(issue_dir.glob(f"original_{base_name}_page_*.txt"))
    
    for orig_file, new_file in zip(original_files, page_files):
        try:
            with open(orig_file, 'r', encoding='utf-8') as f:
                original_text = f.read()
            with open(new_file, 'r', encoding='utf-8') as f:
                new_text = f.read()
            
            row = TableRow()
            
            # Original text cell
            cell = TableCell()
            p = P(text=original_text)
            cell.addElement(p)
            row.addElement(cell)
            
            # New OCR cell
            cell = TableCell()
            p = P(text=new_text)
            cell.addElement(p)
            row.addElement(cell)
            
            table.addElement(row)
        except Exception as e:
            logging.error(f"Error creating comparison row: {str(e)[:200]}")
    
    doc.text.addElement(table)
    output_path = issue_dir / f"{base_name}_comparison.odt"
    doc.save(str(output_path))
    return output_path

def process_pdf(pdf_path, output_dir):
    try:
        base_name = Path(pdf_path).stem
        issue_dir = Path(output_dir) / base_name
        issue_dir.mkdir(parents=True, exist_ok=True)

        original_texts = []
        
        with fitz.open(pdf_path) as doc:
            total_pages = len(doc)
            if CONFIG['TRIAL_MODE']:
                total_pages = min(3, total_pages)
            
            # First pass: Extract original text
            for page_num in range(total_pages):
                try:
                    page = doc.load_page(page_num)
                    original_text = page.get_text()
                    original_path = issue_dir / f"original_{base_name}_page_{page_num+1}.txt"
                    original_path.write_text(original_text, encoding='utf-8')
                    original_texts.append(original_text)
                except Exception as e:
                    logging.error(f"Error extracting original text page {page_num+1}: {str(e)[:200]}")
                    original_texts.append("")

            # Second pass: Process with OCR
            for page_num in range(total_pages):
                try:
                    page = doc.load_page(page_num)
                    pix = page.get_pixmap()
                    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
                    
                    # Calculate DPI
                    pdf_width_pt = page.rect.width
                    pdf_height_pt = page.rect.height
                    dpi_x = pix.width / (pdf_width_pt/72)
                    dpi_y = pix.height / (pdf_height_pt/72)
                    avg_dpi = int((dpi_x + dpi_y) / 2)

                    # Preprocess image
                    preprocessed = preprocess_image(np.array(img))
                    
                    # Save preprocessed image if enabled
                    preprocessed_path = None
                    if CONFIG['SAVE_PREPROCESSED_IMAGES']:
                        preprocessed_path = issue_dir / f"{base_name}_preprocessed_page_{page_num+1}.png"
                        Image.fromarray(preprocessed).save(str(preprocessed_path), dpi=(avg_dpi, avg_dpi))

                    # Build OCR command
                    cmd = [
                        'ocrmypdf',
                        '-l', CONFIG['LANGUAGES'],
                        '--force-ocr',
                        '--optimize', str(CONFIG['OPTIMIZE_LEVEL']),
                        '--tesseract-timeout', str(CONFIG['TESSERACT_TIMEOUT']),
                        '--image-dpi', str(avg_dpi),
                        str(pdf_path),
                        str(issue_dir / "temp_ocr_output.pdf")
                    ]
                    
                    subprocess.run(cmd, check=True, timeout=CONFIG['TESSERACT_TIMEOUT'])
                    
                    # Extract new OCR text
                    with fitz.open(issue_dir / "temp_ocr_output.pdf") as ocr_doc:
                        new_text = ocr_doc.load_page(page_num).get_text()
                        new_text_path = issue_dir / f"{base_name}_page_{page_num+1}.txt"
                        new_text_path.write_text(new_text, encoding='utf-8')

                    # Cleanup temporary files
                    if preprocessed_path and not CONFIG['SAVE_PREPROCESSED_IMAGES']:
                        preprocessed_path.unlink(missing_ok=True)
                        
                    logging.info(f"Processed page {page_num+1}/{total_pages}")

                except Exception as page_error:
                    logging.error(f"Error processing page {page_num+1}: {str(page_error)[:200]}")
                    continue

            # Create comparison document after processing all pages
            if CONFIG['COMPARISON_DOC']:
                comparison_path = create_comparison_doc(issue_dir, base_name)
                logging.info(f"Created comparison document: {comparison_path}")

            # Final cleanup
            (issue_dir / "temp_ocr_output.pdf").unlink(missing_ok=True)

    except Exception as doc_error:
        logging.error(f"Error processing document: {str(doc_error)[:200]}")

def process_pdfs(input_dir, output_dir):
    input_path = Path(input_dir)
    for pdf_file in input_path.glob('*.pdf'):
        process_pdf(pdf_file, output_dir)

if __name__ == "__main__":
    logging.info("Starting OCR processing with comparison feature")
    process_pdfs('input_pdfs', 'output_text')
```

