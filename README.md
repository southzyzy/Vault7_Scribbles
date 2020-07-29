# Scribbles
A document-watermarking preprocessing system to embed "Web beacon"-style tags into documents that are likely to be copied by Insiders, Whistleblowers, Journalists or others.

## Installation

### Test Environment
| Platform / Version | Source Link | 
|:---:|:---:|
| Windows 10 Pro 	| <a href="https://www.microsoft.com/en-au/software-download/windows10ISO"> `Windows 10 ISO` </a>	|
| Visual Studio     | <a href="https://visualstudio.microsoft.com/"> `Visual Studio` </a> |
| Python 3.8		| <a href="https://visualstudio.microsoft.com/"> `Python 3.8` </a> |
| MS Office 2013 / 2016 | `N.A` |

### Clone
- Download the <a href="https://wikileaks.org/vault7/document/Scribbles/"> `Scribbles Source Code` </a>

### Build the Scribbles Executable
1. Change directory to `Scribbles_src/` and open `Scribbles.sln` with Visual Studio.
2. On the `Solution Explorer`, right-click and `Build Solution` project.
<img src="https://imgur.com/PCgqmA7.png" width="300" height="400" />

3. After the solution have been built, `bin/Debug/` directory should be created where it contains `Scribbles.exe`.

---

## Running Scribbles
This program requires the following prerequisite to run smoothly:
1. Ensure `MS Office Version 97 - 2016` is installed. 
2. Create `InputDir` in `bin/Debug/` folder.
3. Place documents to be watermarked inside sub folders within `InputDir`, `bin/Debug/InputDir/SubFolder/Impt_Doc`.
4. Open cmd on `bin/Debug/` directory.
5. Configure `HostServerNameList` in `ScribblesOutput_exampleParams.xml` to `127.0.0.1:8080`.
> Follow the below screenshot for more information.
    
<img src="https://imgur.com/nHsyJeR.png" height="250" />
                                                                                             
6. Run scribbles using this command:
```commandline
.\Scribbles.exe --inputReceiptFile ScribblesConfig_exampleParams.xml –inputDir=.\InputDir --outputDir=.\OutputDir >> ScribblesOutput.txt
```
7. Watermarked files will be outputted to `bin/Debug/OutputDir/`.
8. Set up `simple HTTP.Server` using Python: `python -m http.server 8080`.
> You can also use this Python HTTP Server script to see a more detailed log data.
> Using the script below require some changes to be done on `WatermarkLog.tsv` see under `Notes` to format `WatermarkLog.tsv`. 
```script
#!/usr/bin/env python3
"""
Very simple HTTP server in python for logging requests
Usage::
    ./server.py [<port>]
"""
from http.server import BaseHTTPRequestHandler, HTTPServer
import logging
import pandas as pd
from datetime import datetime

pd.options.display.max_colwidth = 100

# Watermark Database
WATERMARK_DB = r".\WatermarkLog.tsv"
watermark_df = pd.read_csv(WATERMARK_DB, sep=r'\s+')


class S(BaseHTTPRequestHandler):
    def _set_response(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()

    def do_GET(self):
        watermark_tag = self.path.split('/')[3]
        dt_string = datetime.now().strftime("%d/%m/%Y %H:%M:%S")
        file_triggered = watermark_df.loc[watermark_df['Formatted_Watermark_Tag'] == watermark_tag].File_Input_Path.to_string(index=False).lstrip()
        logging.info("GET request,\nDate Time: %s\nFile Opened: %s\nPath: %s\nHeaders:\n%s\n", dt_string, file_triggered, str(self.path), str(self.headers))

        self._set_response()
        self.wfile.write("GET request for {}".format(self.path).encode('utf-8'))

    def do_POST(self):
        content_length = int(self.headers['Content-Length']) # <--- Gets the size of data
        post_data = self.rfile.read(content_length) # <--- Gets the data itself
        logging.info("POST request,\nPath: %s\nHeaders:\n%s\n\nBody:\n%s\n",
                str(self.path), str(self.headers), post_data.decode('utf-8'))

        self._set_response()
        self.wfile.write("POST request for {}".format(self.path).encode('utf-8'))

def run(server_class=HTTPServer, handler_class=S, port=8080):
    logging.basicConfig(level=logging.INFO)
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    logging.info('Starting httpd...\n')
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass
    httpd.server_close()
    logging.info('Stopping httpd...\n')

if __name__ == '__main__':
    from sys import argv

    if len(argv) == 2:
        run(port=int(argv[1]))
    else:
        run()
```
                                                                    
9. Open watermarked docs in `OutputDir` and observe `http.server` response.

---

## Documentation

### Static Analysis
The tool is for “pre-generating watermarks and inserting those watermarks into documents that are apparently being stolen by FIO (Foreign Intelligence Officers) actors.”

Scribbles and similar tools such as web beacons are used by organizations to determine questions like: Did the document leak? Where was it opened? Who was the owner of the document that was opened? When was it opened?

According to WikiLeaks, Scribble works exclusively with Microsoft Office documents. The tool, according to the user guide has been “successfully tested” to work with Microsoft Office 2013 (on Windows 8.1 x64) and Office 97-2016 running on Windows 98 and above.

Customers using Office 365 or Office 2013 and newer are protected by default, as these documents will open in Protected View, which blocks network access,

WikiLeaks’ copy of the CIA’s Scribbles user manual says the tool will not work on encrypted or password-protected documents. The CIA also warns that if a document with a Scribbles’ watermark is opened in an alternative document viewing program, such as OpenOffice or LibreOffice, it may result in revealing watermarks and URLs for the user.

> Read More: <a href="https://wikileaks.org/vault7/#Scribbles"> WikiLeaks#Scribbles </a>  | <a href="https://threatpost.com/wikileaks-reveals-cia-tool-scribbles-for-document-tracking/125299/"> ThreatPost: WikiLeaks Reveals CIA Tool ‘Scribbles’ For Document Tracking </a>

### Dynamic Analysis
Dynamic analysis shows that upon execution, the executable will first check if the file/PID exists in the watermark log .\WatermarkLog.tsv, user will be promted if the file has already been watermarked before and will have the option to either Remake, Create new watermark or Skip the file.

The executable will then upzip the contents of the documents and insert the beacon in a form of a watermark image and repack it, outputing the file to ./OutputDir

Each watermarked document is tagged with a PID and stored in the WatermarkLog.tsv which contains 
1. File_Input_Host
2. File_Input_Path
3. File_Input_Hash
4. Watermark_TimeStamp
5. Watermark_Tag / PID

On the C2 server side, once the watermarked document is opened, the C2 will recieve a GET request trying to get a randomly generated file in a directory which can be set in the 'ScribblesOutput_exampleParams.xml'

The information that can be retrieved from this GET request are
1. Watermark PID
2. Host IP
3. Timestamp

But we have also proven that more information can be retrieved, in this case we used a modified http.server.py and recieved the following additional information
1. Windows Version
2. MS Office Version
3. Name of file opened (By checking the PID and pulling the information from 'watermarklog.tsv')
4. Accessed Date and Time

Lastly would be that amongst the 3 office file extentions (.doc, .xlxs, .pptx), when watermarking a .pptx document a prompt will be shown stating that the user will have to click enable editing by default while the other 2 documents will show up as 'Read-Only' plain text view when opened from a different host than the one who created the document.
The beacons in both cases will not activate unless the user clicks on the 'Enable Editing' in which will activate the beacon and call back to the C2


---

## Notes
`WatermarkLog.tsv` needs some formatting to be done:

### Problems
1. The Column Header Name contains spaces. 
2. The Row Data have inconsistent indentation.
3. The script will fail if the filename of `docx`, `doc`, `docm`, `pptx`, `ppt`, `pptm`, `xls`, `xlsm`, `xlsx`, `xlsb` contains spaces. i.e. `Hello World.docx`

> All these makes problems make the data hard to process, so some changes is needed to be done.

### Solution
**Manual Indentation**: We believe that the `WatermarkLog.tsv` is not meant to be used for scripting or automation since it contains inconsitency. The fatest way to resolve this is by reformatting the log file yourself.
1. Remove the `spaces` in Column Header and replace it with `_`.
2. Add/Remove unncessary tabs of indentation that exist in the Row Data.
3. Make sure when the watermarked document filename contains no spaces. i.e. `Hello_World.docx` 

> This will solve the problems if you use our custom HTTPServer script.

The snippet below is an example on how the `WatermarkLog.tsv` should look like after formatting:
<img src="https://imgur.com/KoNXgkh.png" alt="WatermarkLog.tsv" />

---

## Contributors
| **Gerald Peh** | **Tan Zhao Yea** | **Clement Chin** | **Ng Swee Kiat**
| :---: | :---: | :---: | :---: |
| [![Gerald Peh](https://avatars3.githubusercontent.com/u/20138589?v=3&s=200)](https://www.linkedin.com/in/gxraldpeh/)    | <img src="https://imgur.com/jWUo1wl.png" height="200" /> | <img src="https://imgur.com/DdXCu4t.png" height="200" /> | <img src="https://imgur.com/R7YD5JH.png" height="200" />
| <a href="https://github.com/hellogeraldblah" target="_blank">`github.com/hellogeraldpeh`</a> | <a href="http://github.com/southzyzy" target="_blank">`github.com/southzyzy`</a> | <a href="http://github.com/notclement" target="_blank">`github.com/notclement`</a> | <a href="https://github.com/RaptorShark27"> `github.com/RaptorShark27` </a>


