# QTC OCR


sudo apt-get install poppler-utils

### Unpack portfolio

pdfdetach -saveall 1970-tal.pdf

´´´
#!/bin/bash

# Create a directory for the output
mkdir -p ocr_output

# Loop through all PDF files in the current directory
for pdf in *.pdf; do
    # Get the base name of the PDF file (without extension)
    base_name=$(basename "$pdf" .pdf)
    
    # Create a directory for this PDF's output
    mkdir -p "ocr_output/$base_name"
    
    # Get the number of pages in the PDF
    num_pages=$(pdfinfo "$pdf" | grep Pages | awk '{print $2}')
    
    # Process each page separately
    for ((page=1; page<=num_pages; page++)); do
        # Extract single page from PDF
        pdfseparate -f $page -l $page "$pdf" "ocr_output/$base_name/page_${page}.pdf"
        
        # OCR the single page
        ocrmypdf --force-ocr "ocr_output/$base_name/page_${page}.pdf" "ocr_output/$base_name/ocr_page_${page}.pdf"
        
        # Extract text from OCR'd PDF
        pdftotext -layout "ocr_output/$base_name/ocr_page_${page}.pdf" "ocr_output/$base_name/page_${page}.txt"
        
        # Remove temporary PDF files
        rm "ocr_output/$base_name/page_${page}.pdf" "ocr_output/$base_name/ocr_page_${page}.pdf"
    done
done
``

