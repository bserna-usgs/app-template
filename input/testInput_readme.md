# Input 

For testing purposes I created a sample of related research articles in the 'sentences_352' table format. 

### Process

1. Research articles in PDF format: convert image to tiff file format using ImageMagick
2. Convert tiff file into text using Tesseract for ocr. Note: There is possible loss here, no optimization and any figures/tables were not addressed. 
3. A few of the sample articles contained utf-16 characters. This should be able to be addressed with many tools, I wrote a quick script using python. 
4. Once the input files are prepared they are ready to be processed through the Stanford CoreNLP tool (v3.5.2). I output the results in TSV format using the contll parameter. 
5. After the NLP output is generated the files were reformatted into the schema for the sentences_352 table. Then concatenated into one file and loaded into a test database in postgresql.  