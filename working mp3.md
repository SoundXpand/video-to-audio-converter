<!DOCTYPE html>
<html>
<head>
  <title>Video to Audio Converter</title>
</head>
<body>
  <h1>Video to Audio Converter</h1>
  <input type="file" id="videoInput">
  <select id="formatSelect">
    <option value="mp3">MP3</option>
    <option value="wav">WAV</option>
    <option value="flac">FLAC</option>
    <option value="ogg">OGG</option>
    <option value="m4a">M4A</option>
    <option value="wma">WMA</option>
    <option value="aac">AAC</option>
    <option value="movpkg">MOVPKG</option>
    <option value="raw">RAW</option>
  </select>
  <button onclick="convertVideo()">Convert</button>
  <div id="progressIndicator"></div>
  <a id="downloadLink" href="" download style="display: none;">Download Converted Audio</a>

  <script>
    function convertVideo() {
      const fileInput = document.getElementById('videoInput');
      const formatSelect = document.getElementById('formatSelect');
      const progressIndicator = document.getElementById('progressIndicator');
      const downloadLink = document.getElementById('downloadLink');

      const file = fileInput.files[0];
      const format = formatSelect.value;

      if (!file) {
        alert('Please select a video file.');
        return;
      }

      progressIndicator.innerHTML = 'Converting...';

      const formData = new FormData();
      formData.append('video', file);
      formData.append('format', format);

      fetch('/convert', {
        method: 'POST',
        body: formData
      })
      .then(response => response.text())
      .then(filename => {
        progressIndicator.innerHTML = '';
        downloadLink.href = `/download/${filename}`;
        downloadLink.style.display = 'block';
      })
      .catch(error => {
        console.error('Conversion error:', error);
        progressIndicator.innerHTML = 'Conversion failed.';
        downloadLink.style.display = 'none';
      });
    }
  </script>
</body>
</html>



const express = require('express');
const multer = require('multer');
const ffmpeg = require('fluent-ffmpeg');
const { v4: uuidv4 } = require('uuid');
const path = require('path');
const fs = require('fs');

const app = express();
const upload = multer({ dest: 'uploads/' });
const outputDirectory = path.join(__dirname, 'output');

app.use(express.static('public'));

app.post('/convert', upload.single('video'), (req, res) => {
  if (!req.file) {
    return res.status(400).send('No video file provided.');
  }

  const videoPath = req.file.path;
  const fileId = uuidv4();
  const outputFormats = {
    mp3: 'audio/mpeg',
    wav: 'audio/wav',
    flac: 'audio/flac',
    ogg: 'audio/ogg',
    m4a: 'audio/mp4',
    wma: 'audio/x-ms-wma',
    aac: 'audio/aac',
    movpkg: 'audio/mp4',
    raw: 'audio/x-raw'
  };

  const preferredFormat = req.body.format || 'mp3';

  const outputFileName = fileId + '.' + preferredFormat;
  const outputFilePath = path.join(outputDirectory, outputFileName);

  ffmpeg(videoPath)
    .output(outputFilePath)
    .noVideo()
    .audioCodec('libmp3lame')
    .on('end', () => {
      convertToPreferredFormat(outputFilePath, preferredFormat)
        .then((convertedFilePath) => {
          res.send(path.basename(convertedFilePath));
          cleanupFiles([videoPath, outputFilePath]);
        })
        .catch((error) => {
          console.error('Conversion error:', error);
          res.status(500).send('Conversion failed.');
          cleanupFiles([videoPath, outputFilePath]);
        });
    })
    .on('error', (error) => {
      console.error('Conversion error:', error);
      res.status(500).send('Conversion failed.');
      cleanupFiles([videoPath]);
    })
    .run();
});

app.get('/download/:filename', (req, res) => {
  const fileName = req.params.filename;
  const filePath = path.join(outputDirectory, fileName);

  if (fs.existsSync(filePath)) {
    res.download(filePath, fileName, (error) => {
      if (error) {
        console.error('Download error:', error);
        res.status(500).send('Download failed.');
      } else {
        cleanupFiles([filePath]);
      }
    });
  } else {
    res.status(404).send('File not found.');
  }
});

function cleanupFiles(filePaths) {
  for (const filePath of filePaths) {
    fs.unlink(filePath, (error) => {
      if (error) {
        console.error(`Failed to delete file: ${filePath}`, error);
      }
    });
  }
}

function convertToPreferredFormat(inputFile, preferredFormat) {
  return new Promise((resolve, reject) => {
    const outputFileName = uuidv4() + '.' + preferredFormat;
    const outputPath = path.join(outputDirectory, outputFileName);

    ffmpeg(inputFile)
      .output(outputPath)
      .audioCodec(preferredFormat)
      .on('end', () => resolve(outputPath))
      .on('error', (error) => reject(error))
      .run();
  });
}

const port = 3000;
app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
