# QTC OCR


sudo apt-get install poppler-utils

### Unpack decade portfolio

pdfdetach -saveall 1970-tal.pdf

### Build JBIG2 encoder
https://ocrmypdf.readthedocs.io/en/latest/jbig2.html

```
[sudo] apt install autotools-dev automake libtool libleptonica-dev
```
```git clone https://github.com/agl/jbig2enc
cd jbig2enc
./autogen.sh
./configure && make
[sudo] make install
``


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

